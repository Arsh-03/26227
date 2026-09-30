# Geospatial Lakehouse Architecture Specification

**Document ID:** `ARCH-02-GEOSPATIAL-LAKEHOUSE`  
**Classification:** Sovereign Defense & Enterprise Architecture Specification  
**Version:** 2.0.0 (Enhanced Sovereign & Dual-Feasibility Edition)  
**Status:** Approved  
**Target Audience:** Database Engineers, Geospatial Data Architects, Security Engineers, Backend Developers  
**Regulatory Compliance:** National Geospatial Policy 2022 (NGP 2022), MeitY Tier-IV Storage Standards, In-SPACe / RSDP Remote Sensing Mandates, FIPS 140-3 Cryptographic Integrity  

---

## 1. Executive Summary & Architectural Motivation

Traditional Earth Observation (EO) and Geospatial Intelligence (GEOINT) infrastructures suffer from the **"Dual-Silo Dilemma"**:
- **Object Storage Silos:** Store massive raster scenes (GeoTIFFs, NetCDF, HDF5) as opaque binary blobs. Querying or visualization requires downloading multi-gigabyte files across the network, exhausting bandwidth and crippling real-time responsiveness.
- **Relational / Vector Silos:** Maintain metadata, spatial polygons, and AI vector embeddings in separate databases. Executing compound tactical queries (e.g., *"find naval vessels berthed within 10 km of coordinate X captured between July and August with cosine similarity > 0.88"*) requires high-latency, multi-hop application-level joins.

Project GeoSurge unifies multi-sensor rasters, vector geometries, spatiotemporal catalogs, and multi-dimensional AI embeddings into a single **Sovereign Geospatial Lakehouse**. 

By pairing **Cloud-Optimized GeoTIFFs (COGs)**, **STAC v1.0.0 (with ISRO extensions)**, **Uber H3 Discrete Global Grid Systems (DGGS)**, **PostgreSQL 16 with PostGIS 3.4**, and **pgvector 0.7+ (HNSW)**, the platform achieves sub-80ms hybrid queries without data duplication.

```mermaid
flowchart LR
    subgraph INGRESS ["Ingress Zone (Air-Gapped & Diode)"]
        direction TB
        SIP["Sovereign Ingest Packets (SIP)\n(Encrypted NVMe / LTO Tape / Optical Diode)"]
        BHU["ISRO Bhoonidhi / NRSC\n(Cartosat-3, Resourcesat-2A, EOS-04)"]
        OPEN_S2["Copernicus / USGS Feeds\n(Sentinel-1/2, Landsat-8/9)"]
    end

    subgraph STORAGE ["Sovereign Object Storage Tier (MinIO / Ceph)"]
        direction TB
        COG["Cloud-Optimized GeoTIFFs (COGs)\n- 512x512 Blocks | DEFLATE + Predictor 2\n- Embedded Multi-Scale Overviews (2x-32x)"]
        MASKS["Binary Change & Cloud Masks"]
        THUMBS["H3 512x512 Image Chips (WebP)"]
    end

    subgraph LAKEHOUSE_CORE ["PostgreSQL 16 Sovereign Lakehouse Engine"]
        direction TB
        STAC["STAC v1.0.0 Catalogs\n(ISRO + International Datasets)"]
        H3_GRID["Uber H3 DGGS Hexagonal Grid\n(Res 8 Shard Keys & Res 9 Resolution)"]
        VECTORS["pgvector 768-d Float16 Embeddings\n(HNSW Cosine Vector Indexes)"]
        POSTGIS["PostGIS Spatial Geometries\n(GiST Indexes | Multi-CRS: 4326 & LCC 7755)"]
        RLS["NGP 2022 Security & RLS\n(Attribute-Based Access Redaction)"]
    end

    subgraph CONSUMERS ["Downstream Consumers"]
        TITILER["TiTiler Fast Dynamic Tile Server\n(HTTP Byte-Range Reads RFC 7233)"]
        FASTAPI["Search & Retrieval API Gateway\n(Single-Step Hybrid SQL Queries)"]
        ANALYST["Analyst WebGL Workbench (React 19)"]
    end

    INGRESS --> STORAGE
    STORAGE <-->|HTTP Range Requests (206 Partial)| TITILER
    TITILER --> CONSUMERS
    LAKEHOUSE_CORE <--> FASTAPI
    FASTAPI --> CONSUMERS
```

---

## 2. Object Storage Tier & Bucket Hierarchy

The object storage tier operates identically on single-node local workstations (via standalone MinIO) and multi-node defense data centers (via distributed MinIO or Ceph with Erasure Coding `EC:4`).

```
s3://geosurge-lakehouse/
├── staging/                              # Ephemeral landing zone
│   └── {source}/{year}/{month}/{raw_package_id}.tar.gz
├── cogs/                                 # Golden Cloud-Optimized GeoTIFFs
│   ├── isro-cartosat3/                   # Sub-meter optical (0.25m - 0.8m)
│   │   └── {sub_satellite_track}/{strip_id}/{date}/{item_id}_cog.tif
│   ├── isro-resourcesat2a/               # LISS-4 (5.8m) & AWiFS (56m)
│   │   └── {path}/{row}/{date}/{item_id}_cog.tif
│   ├── isro-eos04-sar/                   # C-Band SAR backscatter (VV/VH)
│   │   └── {orbit}/{frame}/{date}/{item_id}_sar_cog.tif
│   ├── sentinel-2/                       # Multispectral optical (10m)
│   │   └── {mgrs_tile}/{year}/{month}/{item_id}_cog.tif
│   └── indices/                          # Derived analytical bands
│       └── {item_id}_ndvi_ndwi_cog.tif
├── chips/                                # Extracted 512x512 inference chips
│   └── {h3_res8_index}/{chip_uuid}.webp
├── masks/                                # Bi-temporal change masks (GeoTIFF)
│   └── {change_event_uuid}/change_mask.tif
├── dossiers/                             # HSM-signed intelligence export dossiers
│   └── {dossier_id}.pdf
└── offline-basemaps/                     # Offline Survey of India / OSM vector tiles
    └── india_national_extract.pmtiles   # Single-file offline basemap (PMTiles)
```

### 2.1 Physical COG Byte Layout & HTTP Range Request Optimization
All imagery in `s3://geosurge-lakehouse/cogs/` adheres to strict formatting specifications:
1. **Origin Header Placement:** The TIFF header, Image File Directories (IFDs), and overview offsets are strictly positioned in the first $16\text{ KB}$ of the binary file. TiTiler or GDAL can read metadata in a single initial HTTP range request:
   ```http
   GET /cogs/sentinel-2/43REQ/item_cog.tif HTTP/1.1
   Host: minio.geosurge.internal
   Range: bytes=0-16383
   ```
2. **Internal Tile Tiling ($512 \times 512$):** Rather than standard uncompressed strip scanlines, pixels are stored in discrete $512 \times 512$ blocks. Fetching an analyst's immediate screen view requires downloading only the specific 3–6 tiles intersecting the viewport.
3. **Decimation Levels:** Internal multi-scale overviews are pre-rendered at factors of $2\times, 4\times, 8\times, 16\times, 32\times$, and $64\times$ using area-weighted cubic downsampling.
4. **Compression Profile:** Deflate compression with horizontal differencing (`COMPRESS=DEFLATE`, `PREDICTOR=2`), achieving a $58-64\%$ reduction in storage size while maintaining sub-millisecond per-tile decompression speeds.

---

## 3. SpatioTemporal Asset Catalog (STAC v1.0.0) with ISRO Extensions

Every ingested scene is cataloged using the open **STAC v1.0.0** specification, augmented with extensions for Indian Space Research Organisation (ISRO) constellations and defense security metadata.

### 3.1 Sovereign STAC Item JSON Contract
```json
{
  "type": "Feature",
  "stac_version": "1.0.0",
  "stac_extensions": [
    "https://stac-extensions.github.io/eo/v1.1.0/schema.json",
    "https://stac-extensions.github.io/projection/v1.1.0/schema.json",
    "https://geosurge.gov.in/stac/isro-extension/v1.0.0/schema.json",
    "https://geosurge.gov.in/stac/defense-clearance/v1.0.0/schema.json"
  ],
  "id": "ISRO_CS3_20260930_ST43_PANCHRO_008921",
  "collection": "isro-cartosat3-cogs",
  "geometry": {
    "type": "Polygon",
    "coordinates": [[[77.102, 28.704], [77.305, 28.704], [77.305, 28.520], [77.102, 28.520], [77.102, 28.704]]]
  },
  "bbox": [77.102, 28.520, 77.305, 28.704],
  "properties": {
    "datetime": "2026-09-30T05:42:18Z",
    "platform": "cartosat-3",
    "instruments": ["pan-highres"],
    "constellation": "isro-cartosat",
    "gsd": 0.28,
    "eo:cloud_cover": 2.1,
    "view:sun_azimuth": 138.4,
    "view:sun_elevation": 58.2,
    "proj:epsg": 32643,
    "isro:sensor_mode": "NOMINAL_STRIP",
    "isro:processing_level": "L1G_ORTHO",
    "defense:classification": "RESTRICTED",
    "defense:ngp2022_compliance": true,
    "defense:requires_redaction": false
  },
  "assets": {
    "visual": {
      "href": "s3://geosurge-lakehouse/cogs/isro-cartosat3/ST43/2026/09/CS3_PANCHRO_008921.tif",
      "type": "image/tiff; application=geotiff; profile=cloud-optimized",
      "title": "Sub-Meter Orthorectified COG (0.28m GSD)",
      "roles": ["visual", "data", "overview"]
    },
    "indices": {
      "href": "s3://geosurge-lakehouse/cogs/indices/CS3_008921_indices.tif",
      "type": "image/tiff; application=geotiff; profile=cloud-optimized",
      "title": "Radiometric Indices",
      "roles": ["index"]
    }
  }
}
```

---

## 4. Discrete Global Grid System: Uber H3 DGGS Architecture

### 4.1 Hexagonal Mathematical Foundations
Planetary coordinates mapped onto standard square grids produce uneven neighbor distances: orthogonal cells are distance $d$, while diagonal cells are distance $\sqrt{2}d \approx 1.414d$.

In contrast, **Uber H3 hexagonal tessellation** guarantees that every hexagon has exactly 6 neighbors with identical centroid-to-centroid distances:

$$\forall i \in \{1, \dots, 6\}, \quad \text{dist}(\mathbf{c}_0, \mathbf{c}_i) = \sqrt{3} \cdot r = \text{constant}$$

This property is vital for:
1. **Spatial Convolutions & Clustering:** Hexagonal neighborhoods eliminate spatial bias when clustering targets via Deck.gl or PostGIS.
2. **Deterministic Sharding:** H3 64-bit cell integers serve as physical partition shard keys across the database.

```
+------------+--------------------+---------------------+-----------------------------------+
| H3 Level   | Hexagon Area       | Edge Length         | Operational Lakehouse Function    |
+------------+--------------------+---------------------+-----------------------------------+
| Res 4      | 11,005 km²         | 56.4 km             | Database Macro Partitioning Shard |
| Res 7      | 5.161 km²          | 1,220 m             | Regional Scene Aggregation        |
| Res 8      | 0.737 km² (73.7 ha)| 461 m               | Chip Bounding Box Anchor Key      |
| Res 9      | 0.105 km² (10.5 ha)| 174 m               | Tactical Entity Localization      |
+------------+--------------------+---------------------+-----------------------------------+
```

---

## 5. PostgreSQL 16 Lakehouse Schema (PostGIS + pgvector HNSW)

The database schema unites relational data, spatial geometries across multiple datums, and vector embeddings:

```sql
-- 1. Enable Required Sovereign Extensions
CREATE EXTENSION IF NOT EXISTS postgis;
CREATE EXTENSION IF NOT EXISTS vector;
CREATE EXTENSION IF NOT EXISTS pg_trgm;
CREATE EXTENSION IF NOT EXISTS btree_gist;

-- 2. STAC Collections Table
CREATE TABLE stac_collections (
    collection_id VARCHAR(64) PRIMARY KEY,
    title VARCHAR(255) NOT NULL,
    description TEXT,
    license VARCHAR(64) NOT NULL DEFAULT 'sovereign-government',
    spatial_extent GEOMETRY(Polygon, 4326) NOT NULL,
    temporal_start TIMESTAMPTZ NOT NULL,
    temporal_end TIMESTAMPTZ,
    security_classification VARCHAR(32) NOT NULL DEFAULT 'RESTRICTED',
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- 3. Partitioned STAC Items Table (Partitioned by Year/Month of capture)
CREATE TABLE stac_items (
    item_id VARCHAR(128) NOT NULL,
    collection_id VARCHAR(64) NOT NULL REFERENCES stac_collections(collection_id),
    capture_timestamp TIMESTAMPTZ NOT NULL,
    footprint_wgs84 GEOMETRY(Polygon, 4326) NOT NULL,
    footprint_india_lcc GEOMETRY(Polygon, 7755), -- EPSG:7755 (India LCC for contiguous analysis)
    bbox BOX2D NOT NULL,
    h3_res4_partition BIGINT NOT NULL,          -- H3 coarse partition key
    cloud_cover_pct NUMERIC(5, 2) NOT NULL,
    sun_azimuth NUMERIC(5, 2),
    sun_elevation NUMERIC(5, 2),
    gsd_meters NUMERIC(5, 2) NOT NULL,
    cog_s3_uri TEXT NOT NULL,
    indices_s3_uri TEXT,
    security_classification VARCHAR(32) NOT NULL DEFAULT 'RESTRICTED',
    is_redacted BOOLEAN NOT NULL DEFAULT FALSE,
    properties JSONB NOT NULL DEFAULT '{}',
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    PRIMARY KEY (item_id, capture_timestamp)
) PARTITION BY RANGE (capture_timestamp);

-- Spatial and Temporal Indexes
CREATE INDEX idx_stac_items_footprint ON stac_items USING GIST (footprint_wgs84);
CREATE INDEX idx_stac_items_lcc ON stac_items USING GIST (footprint_india_lcc);
CREATE INDEX idx_stac_items_timestamp ON stac_items (capture_timestamp DESC);
CREATE INDEX idx_stac_items_h3_p ON stac_items (h3_res4_partition);

-- 4. Imagery Chips Table (512x512 Window Chunks Anchored to H3 DGGS)
CREATE TABLE imagery_chips (
    chip_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    item_id VARCHAR(128) NOT NULL,
    capture_timestamp TIMESTAMPTZ NOT NULL,
    h3_res8_index BIGINT NOT NULL,
    h3_res9_index BIGINT NOT NULL,
    chip_bbox GEOMETRY(Polygon, 4326) NOT NULL,
    chip_s3_uri TEXT NOT NULL,
    pixel_width INT NOT NULL DEFAULT 512,
    pixel_height INT NOT NULL DEFAULT 512,
    mean_ndvi NUMERIC(4, 3),
    mean_ndwi NUMERIC(4, 3),
    cloud_occlusion_pct NUMERIC(5, 2) NOT NULL DEFAULT 0.0,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_chips_h3_res8 ON imagery_chips (h3_res8_index);
CREATE INDEX idx_chips_h3_res9 ON imagery_chips (h3_res9_index);
CREATE INDEX idx_chips_bbox ON imagery_chips USING GIST (chip_bbox);

-- 5. Multimodal Vector Embeddings (pgvector HNSW)
CREATE TABLE chip_embeddings (
    chip_id UUID PRIMARY KEY REFERENCES imagery_chips(chip_id) ON DELETE CASCADE,
    model_version VARCHAR(64) NOT NULL DEFAULT 'RemoteCLIP-ViT-L-14',
    embedding vector(768) NOT NULL, -- 768-dimensional normalized unit vector
    extracted_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- HNSW Vector Index: Optimized for Sub-30ms Approximate Nearest Neighbor Retrieval
CREATE INDEX idx_chip_embeddings_hnsw 
ON chip_embeddings 
USING hnsw (embedding vector_cosine_ops)
WITH (m = 16, ef_construction = 64);

-- 6. Bi-Temporal Change Events Table (with Cryptographic Provenance)
CREATE TABLE change_events (
    event_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    t1_item_id VARCHAR(128) NOT NULL,
    t2_item_id VARCHAR(128) NOT NULL,
    t1_timestamp TIMESTAMPTZ NOT NULL,
    t2_timestamp TIMESTAMPTZ NOT NULL,
    boundary GEOMETRY(Polygon, 4326) NOT NULL,
    boundary_india_lcc GEOMETRY(Polygon, 7755),
    area_meters_sq NUMERIC(12, 2) NOT NULL,
    transition_class VARCHAR(64) NOT NULL, -- e.g. 'INFRA_NEW', 'DEFOREST'
    confidence_score NUMERIC(4, 3) NOT NULL,
    sar_verified BOOLEAN NOT NULL DEFAULT FALSE,
    cloud_masked BOOLEAN NOT NULL DEFAULT FALSE,
    validation_status VARCHAR(32) NOT NULL DEFAULT 'PENDING_REVIEW',
    reviewed_by VARCHAR(64),
    reviewed_at TIMESTAMPTZ,
    hsm_signature TEXT,                    -- FIPS 140-3 HSM signature
    provenance_hash VARCHAR(64) NOT NULL,  -- SHA-256 hash of T1, T2, and review state
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_change_events_boundary ON change_events USING GIST (boundary);
CREATE INDEX idx_change_events_conf ON change_events (confidence_score DESC);
CREATE INDEX idx_change_events_status ON change_events (validation_status);
```

### 5.1 NGP 2022 Row-Level Security (RLS) & Dynamic Masking
To prevent unauthorized access to strategic areas under NGP 2022 guidelines:
```sql
-- Enable Row Level Security
ALTER TABLE stac_items ENABLE ROW LEVEL SECURITY;

-- Policy: Only analysts with SECRET/TOP_SECRET clearances can see unredacted sub-meter data
CREATE POLICY defense_clearance_policy ON stac_items
FOR SELECT
TO geosurge_analyst_role
USING (
    CASE 
        WHEN current_setting('request.jwt.claim.clearance', true) IN ('SECRET', 'TOP_SECRET') THEN TRUE
        WHEN gsd_meters >= 1.0 AND is_redacted = FALSE THEN TRUE
        ELSE FALSE
    END
);
```

---

## 6. Single-Step Hybrid Query Mechanics

A key engineering breakthrough of the GeoSurge Lakehouse is its ability to execute **combined spatial, temporal, and semantic vector queries in a single query planner execution plan**, avoiding slow memory merges.

```sql
-- Single-Step SQL: Find 50 imagery chips matching semantic text query vector ($1),
-- falling within the operational boundary polygon ($2), captured in Q3 2026,
-- with cloud cover < 10%, ordered by cosine similarity.
EXPLAIN ANALYZE
SELECT 
    c.chip_id,
    c.item_id,
    c.capture_timestamp,
    c.chip_s3_uri,
    ST_AsGeoJSON(c.chip_bbox)::json AS geojson,
    ROUND((1 - (e.embedding <=> $1::vector))::numeric, 4) AS cosine_similarity
FROM imagery_chips c
JOIN chip_embeddings e ON c.chip_id = e.chip_id
WHERE 
    ST_Intersects(c.chip_bbox, ST_GeomFromText($2, 4326))
    AND c.capture_timestamp BETWEEN '2026-07-01T00:00:00Z' AND '2026-09-30T23:59:59Z'
    AND c.cloud_occlusion_pct < 10.0
ORDER BY e.embedding <=> $1::vector ASC
LIMIT 50;
```

**Query Execution Mechanics:**
1. The query optimizer utilizes the **GiST index** on `c.chip_bbox` and the **B-Tree index** on `c.capture_timestamp` to rapidly prune non-qualifying candidate chips down to a small candidate subset ($< 500$ candidates).
2. The **HNSW index** on `e.embedding` performs an exact cosine distance vector scan over the remaining candidates.
3. Total query execution time: **$< 45\text{ ms}$** over an index of $> 5,000,000$ chips.

---

## 7. Dual-Feasibility Implementation Comparison

```
+-----------------------------------------------------------------------------------------+
| LAKEHOUSE FEASIBILITY SPECTRUM                                                          |
+--------------------------+------------------------------+-------------------------------+
| Attribute                | Profile B: MVP Laptop Spec   | Profile A: Sovereign Cluster  |
+--------------------------+------------------------------+-------------------------------+
| Deployment Envelope      | 1x Laptop / Workstation      | 4-8 Node Distributed Cluster  |
| Storage Engine           | MinIO Single-Container       | Distributed MinIO / Ceph EC:4 |
| Storage Capacity         | 50 GB - 250 GB Local SSD     | 100 TB - 2 PB NVMe/SAS Tier   |
| Database Engine          | PostgreSQL 16 (Docker)       | PostgreSQL 16 (Patroni HA)    |
| Vector Index Scale       | 10,000 - 50,000 Chips        | 50,000,000+ Chips             |
| RAM Allocation           | 1.5 GB RAM total             | 64 GB - 128 GB RAM per node   |
| Multi-CRS Support        | EPSG:4326 & EPSG:3857        | EPSG:4326, 3857 & India LCC   |
| Security & Encryption    | Local LUKS Disk Encryption   | FIPS 140-3 HSM + TLS 1.3 mTLS |
+--------------------------+------------------------------+-------------------------------+
```

### 7.1 Running the MVP Lakehouse with Limited Resources
Developers and evaluators can initialize the entire lakehouse in 60 seconds using Docker Compose:
```yaml
version: '3.8'
services:
  lakehouse-db:
    image: pgvector/pgvector:pg16
    environment:
      POSTGRES_DB: geosurge_lakehouse
      POSTGRES_USER: geosurge_admin
      POSTGRES_PASSWORD: local_dev_secret_password
    ports:
      - "5432:5432"
    volumes:
      - ./pgdata:/var/lib/postgresql/data
    deploy:
      resources:
        limits:
          memory: 1.5G

  minio:
    image: minio/minio:RELEASE.2024-03-30T09-41-56Z
    command: server /data --console-address ":9001"
    environment:
      MINIO_ROOT_USER: minioadmin
      MINIO_ROOT_PASSWORD: minioadminpassword
    ports:
      - "9000:9000"
      - "9001:9001"
    volumes:
      - ./miniodata:/data
    deploy:
      resources:
        limits:
          memory: 512M
```
Total memory required: **$< 2.1\text{ GB RAM}$**, proving complete MVP viability on any laptop!

---

## 8. Storage Unit Economics & Lifecycle Archival

### 8.1 On-Premises Sovereign Storage vs. Public Cloud

```
+-----------------------------------------------------------------------------------------+
| STORAGE ECONOMICS: 50,000 SCENES (~35 TB COGS) OVER 3 YEARS                             |
+------------------------------------+-----------------------------+----------------------+
| Metric                             | Public Cloud (AWS/Azure)    | Sovereign On-Prem    |
+------------------------------------+-----------------------------+----------------------+
| Storage Cost per Month             | $770 / month ($0.022/GB)    | $0 (Hardware owned)  |
| Tile Egress Traffic (15 TB/mo)     | $1,350 / month ($0.09/GB)   | $0 (Local LAN / SCIF)|
| 3-Year Total Running Expense       | ~$76,320 (~₹ 64 Lakhs)      | ~$14,500 (₹ 12 Lakhs)|
+------------------------------------+-----------------------------+----------------------+
| SAVINGS VIA SOVEREIGN ON-PREM      | > 81% COST REDUCTION OVER 3 YEARS                  |
+-----------------------------------------------------------------------------------------+
```

### 8.2 Automated Three-Tier Lifecycle Policy
1. **Tier 1: Hot Tier (NVMe SSD / Local MinIO) — Days 0 to 60:** All full-resolution COGs, 512x512 WebP chips, and active pgvector HNSW indexes for instant analyst triage.
2. **Tier 2: Warm Tier (Enterprise SAS HDD Array) — Days 61 to 365:** Full-res rasters moved to erasure-coded high-density disks; $4\times, 8\times, 16\times$ overviews retained in RAM/NVMe cache. pgvector embeddings remain active.
3. **Tier 3: Cold Sovereign Archive (LTO-9 Tape / Offline Vault) — Year 1+:** Raw satellite archives preserved on offline magnetic tape; STAC metadata and vector embeddings remain permanently searchable in PostgreSQL. COG rasters recalled within $< 4\text{ hours}$ upon request.
