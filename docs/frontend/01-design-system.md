# Frontend Specification: Tactical Geospatial Design System

**Document ID:** `FRONT-01-DESIGN-SYSTEM`  
**Classification:** Frontend UX & Design System Specification  
**Version:** 2.0.0 (Enhanced Sovereign & Dual-Feasibility Edition)  
**Status:** Approved  
**Target Audience:** Frontend Engineers, UI/UX Designers, Accessibility Specialists, Product Managers  
**Regulatory Compliance:** National Geospatial Policy 2022 (NGP 2022), Ministry of Defence UI Security Guidelines, MeitY Web Accessibility Guidelines (GIGW 3.0), WCAG 2.1 AAA  

---

## 1. Design Vision & Operational Ergonomics

The **GeoSurge "AetherOps" Design System** is an enterprise, defense-grade visual language engineered for high-consequence geospatial intelligence, crisis monitoring, and Earth observation operations.

### 1.1 Core Architectural Principles
1. **Low-Fatigue SCIF Dark Mode:** Intelligence analysts operate in low-light command centers and SCIFs (Sensitive Compartmented Information Facilities) for 8–12 hour shifts. The visual hierarchy uses deep slate and abyssal charcoal backgrounds with carefully calibrated luminescence to eliminate ocular strain.
2. **High-Glare Tactical Daylight Mode:** For field units (Border Security Force, NDRF disaster relief teams) operating ruggedized laptops in outdoor direct sunlight, the system provides an **Inverted High-Contrast Daylight Mode** with $> 12:1$ contrast.
3. **100% Self-Contained Offline Asset Bundle:** Zero external font or icon CDN requests. All typography (`Inter`, `JetBrains Mono`) and iconography are bundled locally as static WOFF2/SVG assets, guaranteeing seamless operation in completely air-gapped environments.
4. **Mandatory National Security Classification Banners:** Every screen display features standardized sovereign classification headers and footers compliant with Indian government defense marking directives.
5. **Anti-Leak Canvas Watermarking:** Integrated semi-transparent watermarking displaying the analyst's badge, terminal IP, and Indian Standard Time (IST) timestamp to deter unauthorized smartphone screen photography.

---

## 2. Design Tokens

### 2.1 Color Palette & Semantic System

```
+-----------------------------------------------------------------------------------------+
| COLOR ARCHITECTURE:                                                                     |
| [Abyssal Neutrals: 90%] ===> [Tactical UI Accents: 7%] ===> [Alert Vectors: 3%]         |
+-----------------------------------------------------------------------------------------+
```

#### SCIF Dark Mode Surfaces
| Token Name | Hex Value | Tailwind Class | Semantic Role |
| :--- | :--- | :--- | :--- |
| `surface-abyss` | `#06080F` | `bg-[#06080F]` | Application root canvas background |
| `surface-base` | `#0D111D` | `bg-[#0D111D]` | Sidebars, bottom action docks, modal shells |
| `surface-raised` | `#161D2F` | `bg-[#161D2F]` | Interactive cards, input fields, dropdown menus |
| `surface-overlay` | `#1E283F` | `bg-[#1E283F]` | Hover states, active tabs, floating popovers |
| `border-subtle` | `#1F293D` | `border-[#1F293D]` | Panel dividers and non-interactive borders |
| `border-accent` | `#334155` | `border-[#334155]` | Focused inputs and highlighted card borders |

#### High-Glare Tactical Daylight Mode Surfaces (Field Outdoor Mode)
| Token Name | Hex Value | Tailwind Class | Semantic Role |
| :--- | :--- | :--- | :--- |
| `day-surface-base` | `#FFFFFF` | `bg-white` | Root background for outdoor sunlight readability |
| `day-surface-raised`| `#F1F5F9` | `bg-slate-100`| High-contrast daylight card background |
| `day-text-primary` | `#0F172A` | `text-slate-900`| Deep jet-black text (14:1 contrast ratio) |
| `day-border-accent` | `#475569` | `border-slate-600`| Bold structural card boundaries |

#### Tactical Accents & Geospatial Vectors
| Token Name | Hex Value | Tailwind Class | Semantic Role |
| :--- | :--- | :--- | :--- |
| `accent-cyan` | `#00F0FF` | `text-[#00F0FF]` | Primary buttons, active camera reticle, tool focus |
| `status-confirmed`| `#10B981` | `text-[#10B981]` | Verified changes, high-confidence matches ($\ge 90\%$) |
| `status-warning` | `#F59E0B` | `text-[#F59E0B]` | Moderate confidence ($70-89\%$), pending reviews |
| `status-threat` | `#FF1E44` | `text-[#FF1E44]` | Critical infrastructure changes, demolition alerts |
| `status-marine` | `#00E5FF` | `text-[#00E5FF]` | Water bodies, naval platforms, NDWI overlays |
| `status-radar` | `#A855F7` | `text-[#A855F7]` | C-Band SAR backscatter and coherence overlays |

---

### 2.2 National Security Classification Banner Tokens

Compliant with Ministry of Defence and Government of India classification levels:

| Classification Level | Banner Background | Text Color | Display String |
| :--- | :--- | :--- | :--- |
| **RESTRICTED** | `#10B981` (Emerald) | `#FFFFFF` (White) | `RESTRICTED // GOVT OF INDIA // NGP-2022 COMPLIANT` |
| **CONFIDENTIAL** | `#3B82F6` (Tactical Blue)| `#FFFFFF` (White) | `CONFIDENTIAL // AUTHORIZED DEFENSE PERSONNEL ONLY` |
| **SECRET** | `#F59E0B` (Amber) | `#000000` (Black) | `SECRET // NOFORN // EYES ONLY SOVEREIGN ENCLAVE` |
| **TOP SECRET** | `#DC2626` (Crimson) | `#FFFFFF` (White) | `TOP SECRET // STRICT AIR-GAP ENCLAVE ENFORCED` |

---

### 2.3 Typography Hierarchy (100% Self-Hosted WOFF2)

All fonts are bundled locally within `src/assets/fonts/` with zero Google Fonts CDN requests:

| Role | Font Family | Weight | Size / Line-Height | Tracking | Semantic Usage |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Display Header** | `Inter`, sans-serif | 700 (Bold) | `20px / 28px` | `-0.02em` | Application headers, modal titles |
| **Section Header** | `Inter`, sans-serif | 600 (SemiBold) | `14px / 20px` | `-0.01em` | Sidebar group titles, card headers |
| **Body Standard** | `Inter`, sans-serif | 400 (Regular) | `13px / 18px` | `0em` | Explanatory text, labels, tooltips |
| **Spatial / Numeric**| `JetBrains Mono` | 500 (Medium) | `12px / 16px` | `0.02em` | Coordinates, timestamps, H3 hashes, confidence % |
| **Micro Caption** | `JetBrains Mono` | 400 (Regular) | `10px / 14px` | `0.04em` | Hotkey badges, sensor metadata tags |

---

## 3. Geospatial Symbology & Map Styling Standards

```
+---------------------------------------------------------------------------------+
| MAP STYLING MATRIX                                                              |
+----------------------+--------------------+--------------------+----------------+
| Vector Feature Class | Fill Color / Alpha | Stroke / Width     | Dash Pattern   |
+----------------------+--------------------+--------------------+----------------+
| New Infrastructure   | #FF1E44 (40% Opac) | #FF1E44 (2px Solid)| None           |
| Deforestation        | #FF9F1C (35% Opac) | #FF9F1C (1.5px)    | None           |
| Water Inundation     | #00E5FF (35% Opac) | #00E5FF (1.5px)    | None           |
| Earthworks / Mining  | #FFE600 (35% Opac) | #FFE600 (1.5px)    | None           |
| Pending Review Area  | #FFE600 (20% Opac) | #FFE600 (1.5px)    | [4, 4] Dashed  |
| H3 Hexagon Bounds    | None (0% Opac)     | #334155 (0.8px)    | None           |
| Active Target Reticle| None (0% Opac)     | #00F0FF (2px Solid)| Animated Pulsar|
+----------------------+--------------------+--------------------+----------------+
```

---

## 4. Core Reusable Component Specifications

### 4.1 Tactical Hotkey Button (`<TacticalButton />`)
- Includes primary text label, status indicator dot, and an embedded right-aligned keyboard shortcut badge (`<kbd>`).
- Provides immediate visual feedback: background flashes on hotkey press with a 150ms ripple animation.

```tsx
interface TacticalButtonProps {
  label: string;
  hotkey?: string; // e.g. "A"
  variant: "primary" | "danger" | "neutral" | "tactical-cyan";
  size?: "sm" | "md";
  onClick: () => void;
  disabled?: boolean;
}
```

### 4.2 Split-Swipe Divider Handle (`<SplitDividerHandle />`)
- A vertical high-contrast line (`1px` width) terminating in a centered ergonomic pill handle (`36px \times 20px`).
- Features dual directional chevron icons (`◀ ▶`) and dynamic coordinate telemetry tooltip on hover.
- Reacts smoothly to mouse dragging and keyboard arrows (`Ctrl + Left/Right`).

### 4.3 Classification Banner Header (`<SecurityBanner />`)
- Fixed sticky banner anchored to the top and bottom of the viewport.
- Displays sovereign classification, user badge, and active Indian Standard Time (IST) clock:
  ```
  RESTRICTED // GOVT OF INDIA // NGP-2022 COMPLIANT | ANALYST: OFF-8921 | IST: 20:15:25
  ```

---

## 5. Security, Anti-Leak & Watermarking Specification

### 5.1 Dynamic Canvas Watermark Overlay (`<CanvasWatermark />`)
To prevent unauthorized smartphone photography of classified overhead reconnaissance in command rooms:
- Overlays a dynamic, non-obtrusive semi-transparent watermark across the WebGL map canvas rendered via HTML5 canvas compositing:
  ```
  RESTRICTED // ANALYST-OFF-8921 // IP: 10.14.22.8 // 2026-09-30 20:15:25 IST
  ```
- Repeated across a $45^\circ$ diagonal pattern at $6\%$ opacity: invisible to casual view, clearly legible in any photographic capture.

---

## 6. Dual-Feasibility Implementation Comparison

```
+-----------------------------------------------------------------------------------------+
| DESIGN SYSTEM FEASIBILITY SPECTRUM                                                      |
+--------------------------+------------------------------+-------------------------------+
| Attribute                | Profile B: MVP Laptop Spec   | Profile A: Sovereign Cluster  |
+--------------------------+------------------------------+-------------------------------+
| Build Engine             | Vite + Tailwind CSS v3       | Vite + Tailwind CSS v3        |
| Typography Bundle        | Local WOFF2 (180 KB)         | Local WOFF2 (180 KB)          |
| Asset Storage            | Local `src/assets/` folder   | Air-Gapped NGINX Static Host  |
| Theme Modes              | SCIF Dark Mode               | Dual Mode (SCIF Dark + Day)   |
| Build Time               | < 3 seconds                  | < 3 seconds                   |
| CDN Dependencies         | ZERO (100% Offline Bundle)   | ZERO (100% Offline Bundle)    |
| Accessibility Rating     | WCAG 2.1 AAA Compliant       | WCAG 2.1 AAA Compliant        |
+--------------------------+------------------------------+-------------------------------+
```

---

## 7. Accessibility, Contrast, and Ergonomic Compliance

1. **Ultra-High Contrast Ratios:** All textual elements achieve a minimum contrast ratio of $7.2:1$ against their immediate background in Dark Mode and $> 12:1$ in Daylight Mode, exceeding WCAG 2.1 AAA standards.
2. **Keyboard Traversal & Focus Rings:** The entire user interface is navigable using standard keyboard focus rings with high-visibility cyan focus outlines (`focus-visible:ring-2 focus-visible:ring-[#00F0FF]`).
3. **Motion Reduction Support:** Honors `prefers-reduced-motion: reduce`, disabling camera fly-to animations and replacing them with instant coordinate jumps.
