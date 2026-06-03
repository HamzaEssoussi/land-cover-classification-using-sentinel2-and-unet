# 🛰️ Land Cover Classification using Satellite Imagery

> Classify land cover (forest, water, urban, agriculture…) from satellite imagery using Machine Learning (Random Forest) with Google Earth Engine, QGIS, and Python.

---

## 📋 Table of Contents

- [Overview](#overview)
- [Data Sources](#data-sources)
- [Project Structure](#project-structure)
- [Requirements](#requirements)
- [Installation](#installation)
- [Step-by-Step Workflow](#step-by-step-workflow)
  - [Step 1 — Google Earth Engine Setup](#step-1--google-earth-engine-setup)
  - [Step 2 — Download Satellite Image](#step-2--download-satellite-image)
  - [Step 3 — Collect Training Samples in QGIS](#step-3--collect-training-samples-in-qgis)
  - [Step 4 — Prepare Data in Python](#step-4--prepare-data-in-python)
  - [Step 5 — Train the Model](#step-5--train-the-model)
  - [Step 6 — Classify the Full Image](#step-6--classify-the-full-image)
  - [Step 7 — Visualize the Map](#step-7--visualize-the-map)
- [Results](#results)
- [Evaluation Metrics](#evaluation-metrics)
- [References](#references)

---

## Overview

This project builds a **supervised land cover classification model** from multispectral satellite imagery. Each pixel in the image is assigned one of the following classes:

| Class | Color | Description |
|-------|-------|-------------|
| 🌲 Forest | Dark Green | Dense tree cover |
| 🌾 Agriculture | Yellow | Croplands, farmlands |
| 🏙️ Urban | Gray | Built-up areas, roads |
| 💧 Water | Blue | Rivers, lakes, sea |
| 🟫 Bare Soil | Brown | Exposed soil, deserts |

**Model used:** Random Forest (scikit-learn)  
**Imagery:** Sentinel-2 (10m resolution) via Google Earth Engine

---

## Data Sources

| Source | Resolution | Access | Used For |
|--------|-----------|--------|----------|
| [Sentinel-2 (ESA)](https://sentinel.esa.int) | 10–60 m | Free via GEE | Main imagery |
| [Landsat 8/9 (USGS)](https://earthexplorer.usgs.gov) | 30 m | Free | Alternative imagery |
| [MODIS (NASA)](https://modis.gsfc.nasa.gov) | 250 m–1 km | Free | Global/temporal analysis |
| [NAIP (USDA)](https://www.fsa.usda.gov/programs-and-services/aerial-photography/imagery-programs/naip-imagery/) | 0.6–1 m | Free (USA only) | High-res local analysis |

---

## Project Structure

```
land-cover-classification/
│
├── data/
│   ├── raw/                    # Downloaded satellite images (.tif)
│   ├── samples/                # Training shapefiles from QGIS (.shp)
│   └── processed/              # Extracted pixel features (CSV)
│
├── notebooks/
│   ├── 01_download_gee.ipynb   # Google Earth Engine export
│   ├── 02_prepare_data.ipynb   # Feature extraction
│   ├── 03_train_model.ipynb    # Random Forest training
│   └── 04_classify_map.ipynb   # Full image classification
│
├── src/
│   ├── extract_features.py     # Extract pixel values from shapefile
│   ├── train.py                # Train and save the model
│   ├── predict.py              # Apply model to full image
│   └── evaluate.py             # Accuracy metrics & confusion matrix
│
├── outputs/
│   ├── classification_map.tif  # Final classified raster
│   └── classification_map.png  # Visual export
│
├── requirements.txt
└── README.md
```

---

## Requirements

### Tools to Install

| Tool | Version | Download |
|------|---------|----------|
| Python | ≥ 3.9 | [python.org](https://python.org) |
| QGIS | ≥ 3.x | [qgis.org](https://qgis.org) |
| Google Earth Engine account | — | [earthengine.google.com](https://earthengine.google.com) |

### Python Libraries

```txt
rasterio>=1.3
geopandas>=0.13
numpy>=1.24
pandas>=2.0
scikit-learn>=1.3
matplotlib>=3.7
earthengine-api>=0.1.370
```

---

## Installation

```bash
# 1. Clone the repository
git clone https://github.com/your-username/land-cover-classification.git
cd land-cover-classification

# 2. Create a virtual environment
python -m venv venv
source venv/bin/activate        # Linux/Mac
venv\Scripts\activate           # Windows

# 3. Install dependencies
pip install -r requirements.txt

# 4. Authenticate Google Earth Engine
earthengine authenticate
```

---

## Step-by-Step Workflow

### Step 1 — Google Earth Engine Setup

1. Create a free account at [earthengine.google.com](https://earthengine.google.com)
2. Go to [code.earthengine.google.com](https://code.earthengine.google.com)
3. Draw your **Region of Interest (ROI)** directly on the map using the rectangle tool

---

### Step 2 — Download Satellite Image

Run this script in the **GEE JavaScript Code Editor** to export a cloud-free Sentinel-2 composite to your Google Drive:

```javascript
// ── Google Earth Engine export script ──────────────────────────────────────

// 1. Define your region of interest (drawn on the map or set manually)
var roi = ee.Geometry.Rectangle([8.0, 36.0, 10.5, 37.5]); // Example: Tunisia

// 2. Load Sentinel-2 Surface Reflectance collection
var collection = ee.ImageCollection("COPERNICUS/S2_SR_HARMONIZED")
  .filterDate("2023-06-01", "2023-09-30")   // Summer → less clouds
  .filterBounds(roi)
  .filter(ee.Filter.lt("CLOUDY_PIXEL_PERCENTAGE", 10));

// 3. Create a cloud-free median composite
var image = collection.median().clip(roi);

// 4. Select useful bands + compute NDVI
var bands = image.select(["B2","B3","B4","B8","B11","B12"]); // Blue, Green, Red, NIR, SWIR1, SWIR2
var ndvi  = image.normalizedDifference(["B8","B4"]).rename("NDVI");
var ndwi  = image.normalizedDifference(["B3","B8"]).rename("NDWI");
var ndbi  = image.normalizedDifference(["B11","B8"]).rename("NDBI");
var final = bands.addBands([ndvi, ndwi, ndbi]);

// 5. Export to Google Drive
Export.image.toDrive({
  image: final,
  description: "sentinel2_composite",
  folder: "GEE_exports",
  fileNamePrefix: "sentinel2_roi",
  scale: 10,
  region: roi,
  maxPixels: 1e13
});
```

> 💡 After running, go to **Tasks** tab (top right) → click **Run** to start the export. The `.tif` file will appear in your Google Drive.

---

### Step 3 — Collect Training Samples in QGIS

1. **Open QGIS** and load your `.tif` image via `Layer > Add Layer > Add Raster Layer`
2. Create a new **Shapefile layer**: `Layer > Create Layer > New Shapefile Layer`
   - Geometry type: `Polygon`
   - Add a field: `class` (type: String) or `class_id` (type: Integer)
3. **Draw polygons** on representative areas:

| Class | class_id | How to identify on image |
|-------|----------|--------------------------|
| Forest | 1 | Dark green, textured areas |
| Agriculture | 2 | Light green/yellow, regular patterns |
| Urban | 3 | Gray, geometric structures |
| Water | 4 | Very dark blue/black areas |
| Bare Soil | 5 | Light brown, no vegetation |

4. Draw **at least 20–30 polygons per class** for good accuracy
5. Export as `samples.shp` into `data/samples/`

---

### Step 4 — Prepare Data in Python

```python
# src/extract_features.py

import numpy as np
import pandas as pd
import rasterio
import geopandas as gpd
from rasterio.mask import mask

def extract_pixel_values(image_path, shapefile_path):
    """Extract pixel values from satellite image for each training polygon."""

    gdf   = gpd.read_file(shapefile_path)
    src   = rasterio.open(image_path)

    # Reproject shapefile to match image CRS if needed
    gdf = gdf.to_crs(src.crs)

    all_pixels = []

    for _, row in gdf.iterrows():
        geom     = [row.geometry.__geo_interface__]
        out_img, _ = mask(src, geom, crop=True)           # shape: (bands, H, W)
        pixels   = out_img.reshape(out_img.shape[0], -1).T  # shape: (N_pixels, bands)

        # Remove nodata pixels
        valid = ~np.any(pixels == src.nodata, axis=1)
        pixels = pixels[valid]

        # Add class label
        labels = np.full((pixels.shape[0], 1), row["class_id"])
        all_pixels.append(np.hstack([pixels, labels]))

    data = np.vstack(all_pixels)
    band_names = [f"band_{i+1}" for i in range(src.count)]
    df = pd.DataFrame(data, columns=band_names + ["class_id"])

    df.to_csv("data/processed/training_data.csv", index=False)
    print(f"✅ Extracted {len(df)} pixels from {len(gdf)} polygons")
    return df

# Run
df = extract_pixel_values(
    image_path="data/raw/sentinel2_roi.tif",
    shapefile_path="data/samples/samples.shp"
)
print(df.head())
print(df["class_id"].value_counts())
```

---

### Step 5 — Train the Model

```python
# src/train.py

import pandas as pd
import joblib
from sklearn.ensemble import RandomForestClassifier
from sklearn.model_selection import train_test_split
from sklearn.metrics import classification_report, accuracy_score

# ── Load data ──────────────────────────────────────────────────────────────
df = pd.read_csv("data/processed/training_data.csv")

X = df.drop("class_id", axis=1).values   # Features (band values)
y = df["class_id"].values                 # Labels

# ── Split train / test ─────────────────────────────────────────────────────
X_train, X_test, y_train, y_test = train_test_split(
    X, y, test_size=0.2, random_state=42, stratify=y
)

# ── Train Random Forest ────────────────────────────────────────────────────
model = RandomForestClassifier(
    n_estimators=200,
    max_depth=None,
    min_samples_split=5,
    random_state=42,
    n_jobs=-1              # Use all CPU cores
)
model.fit(X_train, y_train)

# ── Evaluate ───────────────────────────────────────────────────────────────
y_pred = model.predict(X_test)

print(f"Overall Accuracy : {accuracy_score(y_test, y_pred):.4f}")
print("\nDetailed Report:")
print(classification_report(
    y_test, y_pred,
    target_names=["Forest","Agriculture","Urban","Water","Bare Soil"]
))

# ── Save model ─────────────────────────────────────────────────────────────
joblib.dump(model, "outputs/random_forest_model.joblib")
print("✅ Model saved to outputs/random_forest_model.joblib")
```

---

### Step 6 — Classify the Full Image

```python
# src/predict.py

import numpy as np
import rasterio
import joblib
from rasterio.transform import from_bounds

def classify_image(image_path, model_path, output_path):
    """Apply trained model to entire satellite image."""

    model = joblib.load(model_path)

    with rasterio.open(image_path) as src:
        data      = src.read()                              # (bands, H, W)
        meta      = src.meta.copy()
        H, W      = data.shape[1], data.shape[2]

        # Reshape to (N_pixels, bands)
        pixels    = data.reshape(data.shape[0], -1).T       # (H*W, bands)

        # Handle nodata
        nodata_mask = np.any(data == src.nodata, axis=0).flatten()

        # Predict
        print(f"🔍 Classifying {H}×{W} = {H*W:,} pixels…")
        predictions = np.zeros(H * W, dtype=np.uint8)
        valid_pixels = pixels[~nodata_mask]
        predictions[~nodata_mask] = model.predict(valid_pixels)

        # Reshape back to image
        classification_map = predictions.reshape(H, W)

    # Save output raster
    meta.update({"count": 1, "dtype": "uint8", "nodata": 0})

    with rasterio.open(output_path, "w", **meta) as dst:
        dst.write(classification_map.astype(np.uint8), 1)

    print(f"✅ Classification map saved to {output_path}")

# Run
classify_image(
    image_path="data/raw/sentinel2_roi.tif",
    model_path="outputs/random_forest_model.joblib",
    output_path="outputs/classification_map.tif"
)
```

---

### Step 7 — Visualize the Map

```python
# Quick visualization with matplotlib

import numpy as np
import rasterio
import matplotlib.pyplot as plt
import matplotlib.patches as mpatches
from matplotlib.colors import ListedColormap

# Class colors and names
colors     = ["#1a9641", "#ffffbf", "#808080", "#2166ac", "#d7b591"]
class_names = ["Forest", "Agriculture", "Urban", "Water", "Bare Soil"]
cmap       = ListedColormap(colors)

with rasterio.open("outputs/classification_map.tif") as src:
    classification = src.read(1)

fig, ax = plt.subplots(1, 1, figsize=(12, 10))
im = ax.imshow(classification, cmap=cmap, vmin=1, vmax=5)

# Legend
patches = [mpatches.Patch(color=colors[i], label=class_names[i]) for i in range(5)]
ax.legend(handles=patches, loc="lower right", fontsize=11, framealpha=0.9)

ax.set_title("Land Cover Classification — Sentinel-2", fontsize=14, fontweight="bold")
ax.axis("off")
plt.tight_layout()
plt.savefig("outputs/classification_map.png", dpi=150, bbox_inches="tight")
plt.show()
print("✅ Map saved to outputs/classification_map.png")
```

> 💡 You can also open `classification_map.tif` directly in **QGIS**, apply a color palette, and export a professional map.

---

## Results

| Class | Pixels | % Coverage |
|-------|--------|-----------|
| Forest | — | — |
| Agriculture | — | — |
| Urban | — | — |
| Water | — | — |
| Bare Soil | — | — |

> *(Fill this table after running the classification on your area of study)*

---

## Evaluation Metrics

| Metric | Description | Target |
|--------|-------------|--------|
| **Overall Accuracy** | % of correctly classified pixels | > 85% |
| **Kappa Coefficient** | Agreement beyond chance (0–1) | > 0.80 |
| **F1-score per class** | Precision × Recall balance | > 0.80 |

### Example Confusion Matrix output

```
                Predicted
              F    A    U    W    S
Actual  F  [ 95    2    1    0    2 ]
        A  [  3   88    4    0    5 ]
        U  [  1    3   91    0    5 ]
        W  [  0    0    0   99    1 ]
        S  [  2    4    3    1   90 ]

Overall Accuracy : 0.926
Kappa            : 0.906
```

---

## References

- [Google Earth Engine Documentation](https://developers.google.com/earth-engine)
- [Sentinel-2 User Guide — ESA](https://sentinel.esa.int/web/sentinel/user-guides/sentinel-2-msi)
- [scikit-learn Random Forest](https://scikit-learn.org/stable/modules/generated/sklearn.ensemble.RandomForestClassifier.html)
- [rasterio Documentation](https://rasterio.readthedocs.io)
- [QGIS Documentation](https://docs.qgis.org)
- Breiman, L. (2001). *Random Forests*. Machine Learning, 45, 5–32.

---

## License

This project is licensed under the MIT License — see [LICENSE](LICENSE) for details.

---

<div align="center">
  Made with ❤️ | Satellite imagery · Machine Learning · Geospatial Python
</div>
