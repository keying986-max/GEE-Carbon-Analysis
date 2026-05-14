## NPP Trend and Stability Analysis

This repository also includes a Google Earth Engine script for analyzing long-term NPP trends and ecosystem stability using MODIS MOD17A2HGF.

### Purpose

Net primary productivity is an important indicator of ecosystem productivity and carbon sequestration potential. This script calculates annual NPP from MODIS 8-day products and evaluates its temporal trend and interannual stability over a user-defined study area.

### Dataset

| Item | Description |
|---|---|
| Dataset | MODIS/061/MOD17A2HGF |
| Band | PsnNet |
| Temporal resolution | 8-day |
| Analysis period | User-defined |
| Platform | Google Earth Engine Code Editor |

### Methods

1. Load MODIS MOD17A2HGF NPP images.
2. Calculate annual NPP by summing 8-day `PsnNet` images.
3. Estimate pixel-wise linear trend slope.
4. Evaluate trend significance using Pearson correlation p-value.
5. Classify trend direction and significance.
6. Calculate coefficient of variation to describe ecosystem stability.
7. Export GeoTIFF results to Google Drive.

### Outputs

| Output | Description |
|---|---|
| `S` | Linear trend slope of annual NPP |
| `P` | Pearson correlation p-value |
| `CLASS` | Trend significance classification |
| `CV` | Coefficient of variation of annual NPP |
| `CV_CLASS` | Ecosystem stability classification |
| `N` | Number of valid annual observations |

### Trend Classification

| Class | Meaning |
|---|---|
| 1 | Significant increase |
| 2 | Significant decrease |
| 3 | Non-significant trend or insufficient valid observations |

### Stability Classification

| CV class | Meaning |
|---|---|
| 1 | CV < 0.1 |
| 2 | 0.1 ≤ CV < 0.2 |
| 3 | 0.2 ≤ CV < 0.3 |
| 4 | CV ≥ 0.3 |

### How to Use

Open the script in the Google Earth Engine Code Editor and modify the following parameters:

```javascript
studyAreaAsset: 'users/your_username/your_asset_name',
startYear: 2001,
endYear: 2020
