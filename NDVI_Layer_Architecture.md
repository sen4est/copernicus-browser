# NDVI Layer Architecture in Copernicus Browser

## Overview

NDVI layers in the Copernicus Browser use a sophisticated hybrid approach that combines API selection with blob-based rendering, rather than traditional WMS layers. This architecture enables real-time evalscript execution while maintaining tile-based performance.

## API Selection Hierarchy

The system uses an intelligent API selection mechanism with the following priority:

```javascript
// From sentinelhubLeafletLayer.jsx:392-396
const apiType = layer.supportsApiType(ApiType.PROCESSING)
  ? ApiType.PROCESSING           // Preferred for evalscripts
  : layer.supportsApiType(ApiType.WMTS)
  ? ApiType.WMTS                // Fallback for tiled services
  : ApiType.WMS;                // Final fallback
```

### API Types

1. **Processing API (Primary Method)**
   - **ApiType.PROCESSING** is preferred for evalscript-based layers like NDVI
   - Uses Sentinel Hub's Processing API (`/api/v1/process`)
   - Sends evalscript directly in request body
   - Returns processed imagery as blobs

2. **WMTS (Web Map Tile Service)**
   - Fallback for pre-tiled services
   - Better performance for static content

3. **WMS (Web Map Service)**
   - Final fallback option
   - Legacy support for older configurations

## NDVI Evalscript Implementation

NDVI layers absolutely require evalscripts to generate results. The raw satellite data only contains individual spectral bands (red, NIR), and the NDVI visualization is generated in real-time.

### Example NDVI Evalscript

```javascript
//VERSION=3
function setup() {
  return {
    input: [{
      bands: [
        "red",
        "nir",
        "dataMask"
      ]
    }],
    output: [
      { id: "default", bands: 4 },
      { id: "index", bands: 1, sampleType: "FLOAT32" },
      { id: "eobrowserStats", bands: 1 },
      { id: "dataMask", bands: 1 },
    ]
  }
}

function evaluatePixel(samples) {
  let ndvi = (samples.nir - samples.red)/(samples.nir + samples.red);
  const indexVal = samples.dataMask === 1 ? ndvi : NaN;
  let imageVals = valueInterpolate(ndvi,
    [0.0, 0.5, 1.0],
    [
      [1,0,0, samples.dataMask],
      [1,1,0,samples.dataMask],
      [0.1,0.31,0,samples.dataMask],
    ]);

  return {
    default: imageVals,
    index: [indexVal],
    eobrowserStats:[ndvi],
    dataMask: [samples.dataMask],
  };
}
```

## Tile-Based Request Structure

Each map tile is processed individually with geographic bounds:

```javascript
// From sentinelhubLeafletLayer.jsx:410-416
const nwPoint = coords.multiplyBy(tileSize);
const sePoint = nwPoint.add([tileSize, tileSize]);
const nw = L.CRS.EPSG3857.project(this._map.unproject(nwPoint, coords.z));
const se = L.CRS.EPSG3857.project(this._map.unproject(sePoint, coords.z));
const bbox = new BBox(CRS_EPSG3857, nw.x, se.y, se.x, nw.y);
```

### Fixed Tile Size with Variable Ground Resolution

**Key Insight**: NDVI viewing uses a **fixed 512×512 pixel tile system** with **variable ground resolution** based on zoom level:

```javascript
// From sentinelhubLeafletLayer.jsx:240, 351
tileSize: 512,  // Always fixed

const individualTileParams = {
  width: 512,    // Always fixed
  height: 512,   // Always fixed
  zoom: coords.z // Determines geographic area covered
};
```

### Zoom Level Impact on Ground Resolution

**Zoom Level 12 (High Detail):**
- 512×512 pixels covers ~5.12km × 5.12km area
- Each pixel ≈ 10m ground resolution
- Fine-grained NDVI details (individual fields, buildings)

**Zoom Level 8 (Regional View):**
- 512×512 pixels covers ~25.6km × 25.6km area
- Each pixel ≈ 50m ground resolution
- Broader NDVI patterns (regional vegetation trends)

```
Visual Example:

Zoom 12 (High Detail):
┌─────────────┐ 512×512 pixels
│ ████████████ │ = 5.12km × 5.12km area
│ ████████████ │ = 10m per pixel
│ ████████████ │
└─────────────┘

Zoom 8 (Low Detail):
┌─────────────┐ 512×512 pixels
│ ████████████ │ = 25.6km × 25.6km area
│ ████████████ │ = 50m per pixel
│ ████████████ │
└─────────────┘
```

### Important: NDVI Calculation Remains Consistent

The NDVI calculation itself doesn't change with zoom level:
```javascript
let ndvi = (samples.nir - samples.red)/(samples.nir + samples.red);
```

What changes is:
- **Geographic area** each 512×512 tile covers
- **Ground resolution** (meters per pixel)
- **Level of detail** visible in the NDVI visualization
- **Processing cost** vs. **geographic coverage** trade-off

## Blob-Based Rendering

Unlike traditional WMS which uses image URLs, NDVI layers use blob-based rendering:

```javascript
// From sentinelhubLeafletLayer.jsx:424-433
layer.getMap(individualTileParams, apiType, reqConfig).then((blob) => {
  tile.onload = function () {
    URL.revokeObjectURL(tile.src);
    if (onTileImageLoad) {
      onTileImageLoad();
    }
    done(null, tile);
  };
  const objectURL = URL.createObjectURL(blob);
  tile.src = objectURL;
});
```

## Request Flow for NDVI Layers

1. **Tile Request**: Leaflet requests a map tile for specific coordinates
2. **BBox Calculation**: Geographic bounds calculated for tile
3. **API Selection**: Processing API chosen for evalscript support
4. **Processing Request**:
   - Evalscript sent with request
   - Geographic bounds specified
   - Authentication token included
5. **Evalscript Execution**: Server-side processing of NDVI calculation
6. **Blob Response**: Processed image returned as binary blob
7. **Object URL Creation**: Temporary URL created for blob
8. **Tile Rendering**: Image displayed in browser
9. **Cleanup**: Object URL revoked to free memory

## Key Differences from Traditional WMS

### Traditional WMS:
- Static layers served as pre-rendered tiles
- URL-based parameters (LAYERS, STYLES, FORMAT)
- Limited customization
- Fast but inflexible

### NDVI Processing API Approach:
- **Dynamic processing**: Evalscript executed on-demand
- **Blob responses**: Binary image data returned directly
- **Object URLs**: `URL.createObjectURL()` creates temporary URLs for tiles
- **Memory management**: `URL.revokeObjectURL()` cleans up after tile loads

## Layer Creation Process

When an NDVI layer is loaded, the system follows this flow:

1. **Layer Selection**: User selects NDVI visualization from the interface
2. **Layer Creation**: The `createLayer()` function is called with parameters including the evalscript
3. **Evalscript Processing**: The system determines whether to use:
   - **Pre-configured layers** (`createLayerFromService()`) - uses stored evalscripts from configuration
   - **Custom layers** (`createCustomLayer()`) - uses dynamic evalscript for specific datasets
   - **Data fusion layers** (`createDataFusionLayer()`) - combines multiple datasets with evalscripts

## Evalscript Execution Details

### Real-time Computation
- NDVI values are calculated on-demand using the formula `(NIR - Red)/(NIR + Red)`
- Color mapping applied using `valueInterpolate()` function
- Multiple outputs: visualization image, raw index values, and statistics

### Why Evalscripts Are Required
1. **Real-time computation**: NDVI isn't pre-computed - calculated from raw bands
2. **Flexibility**: Different NDVI visualizations can use different color schemes
3. **Band combination**: Combines multiple spectral bands into a single index
4. **Dynamic processing**: Adjusts calculations based on satellite type, quality masks

## Configuration Examples

Many layers in `default_themes.js` show `service: 'WMS'`, but these are **legacy configurations**. The actual implementation dynamically chooses the best API:

- **Pre-configured layers**: May use WMS if evalscripts are pre-stored
- **Custom/Dynamic layers**: Use Processing API for real-time evalscript execution
- **Data fusion layers**: Always use Processing API for multi-dataset combinations

## Authentication and Token Usage

The system supports multiple authentication methods:

### Token Priority for Authenticated Users
```javascript
// From App.jsx:258-271
return auth.user.access_token ?? auth.anonToken;

export const getGetMapAuthToken = (auth) => {
  if (auth.user) {
    const now = new Date().valueOf();
    const isTokenExpired = auth.user.token_expiration < now;

    if (!isTokenExpired) {
      return auth.user.access_token;  // Uses USER token
    }
  }
  return auth.anonToken;  // Falls back to anonymous only if user token expired
};
```

### Authentication Modes
1. **User Authentication**: Uses personal monthly quota when logged in
2. **Anonymous Authentication**: reCAPTCHA-based tokens with limited quota
3. **Fallback Mechanism**: Automatically retries without tokens for public data

## Resolution Management and Chunking

### Two Different Request Systems

The Copernicus Browser uses different approaches for different use cases:

#### 1. Regular Map Viewing (NDVI Display)
- **Always 512×512 pixels** regardless of zoom level
- Used for normal map browsing and NDVI visualization
- Ground resolution varies with zoom level (10m to 50m+ per pixel)
- Optimized for real-time viewing and map navigation

#### 2. Download/Processing Requests (Export, Histogram, etc.)
- **Variable dimensions** up to 2500×2500 pixel limit
- Uses actual calculated dimensions based on geographic area
- Only chunks when either width OR height exceeds 2500 pixels
- Can be any size ≤ 2500×2500 (e.g., 800×600, 1200×900, 2400×1800)

### Chunking Threshold: The 2500-Pixel Rule

```javascript
// From src/const.js:116
export const MAX_SH_IMAGE_SIZE = 2500; // SH services limit

// From Histogram.utils.js:80
if (width <= MAX_SH_IMAGE_SIZE && height <= MAX_SH_IMAGE_SIZE) {
    // Single request - no chunking needed
} else {
    // Chunking required
    const xSplitBy = Math.ceil(width / MAX_SH_IMAGE_SIZE);
    const ySplitBy = Math.ceil(height / MAX_SH_IMAGE_SIZE);
}
```

**Key Finding**: Chunking triggers when **either width OR height exceeds 2500 pixels**, not by total area:

| Dimensions | Total Area | Chunks? | Reason |
|------------|------------|---------|--------|
| 2500×2500 | 6,250,000 | No | Both ≤ 2500 |
| 2600×2000 | 5,200,000 | **Yes** | Width > 2500 |
| 2000×2600 | 5,200,000 | **Yes** | Height > 2500 |
| 1000×1000 | 1,000,000 | No | Both ≤ 2500 |

**Area Paradox**: A smaller total area (2.6M pixels) can trigger chunking while a larger area (6.25M pixels) might not, because the system prioritizes individual dimension limits over total area.

## Performance Benefits

This architecture provides:

1. **Real-time evalscript execution** for complex visualizations
2. **Dynamic parameter adjustment** without redeployment
3. **Complex mathematical operations** (like NDVI calculations)
4. **Efficient memory management** through object URLs
5. **Fallback compatibility** to WMS when Processing API isn't available
6. **Tile-based performance** maintaining map responsiveness
7. **Zoom-adaptive resolution** balancing detail vs. coverage
8. **Intelligent chunking** respecting Sentinel Hub processing limits

## Summary

**NDVI layers use a modern "Processing API + Blob Rendering" approach rather than traditional WMS.** This hybrid architecture enables the flexibility needed for complex visualizations like NDVI while maintaining the performance characteristics of tiled map services. The system is fundamentally different from RGB layers which can use raw band data directly - NDVI requires mathematical processing that only evalscripts can provide.