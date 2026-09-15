# DigitalTwin — Punjab Crop Yield Prediction & Fertilizer Advisory System

A hackathon project that builds a **digital twin for agriculture in Punjab, Pakistan** — a system that lets a farmer or advisor simulate a season before it happens: pick a crop, a district, a sowing date, and a fertilizer plan, and get back a predicted yield, a fertilizer-driven yield uplift, and a live 3D/visual representation of the field in Unity.

The project has two halves:
1. **ML/Remote-Sensing Backend (Python)** — predicts crop yield and fertilizer response from satellite, weather, and soil data.
2. **Unity Front-End** — the visual/interactive "digital twin" layer that consumes the backend's predictions.

---

## 1. What It Does

Punjab's smallholder farmers rarely get to test a decision (which fertilizer blend, when to sow, how much to apply) before committing real money and a real season to it. This system gives them a sandbox:

- Enter **crop + district + sowing date + fertilizer regime**
- Get back:
  - **Baseline yield (Y₀)** — what the field would yield from soil, weather, and satellite vegetation signals alone, with no fertilizer
  - **Fertilizer uplift (ΔY)** — the additional yield from the specific nutrients applied (N, P₂O₅, K₂O, S, Zn), computed from a quadratic diminishing-returns response curve
  - **Final predicted yield**, in both t/ha and **Maunds/Acre** (the unit Punjabi farmers actually use)

It covers **36 districts of Punjab, 46 crops, and 9 agricultural seasons (2016–17 to 2024–25)** of historical data.

---

## 2. How It Works

### Data sources
- **Sentinel-2** multispectral vegetation indices (NDVI, EVI, NDRE, GNDVI, NDWI)
- **Sentinel-1** SAR radar backscatter (VV, VH, VV/VH ratio)
- **ERA5-Land / CHIRPS** gridded weather reanalysis (rainfall, temperature, solar radiation)
- **ISRIC SoilGrids** soil chemistry and physics (pH, organic carbon, clay, sand, moisture)

All pulled via **Google Earth Engine**.

### The core problem it solves
At the time a farmer needs fertilizer advice (pre-season), this season's satellite imagery doesn't exist yet — the crop hasn't grown. So the system runs in one of two modes:
- **Direct Mode** — uses real, already-observed Sentinel-1/2 data when available (past/current seasons)
- **Fallback Mode** — when imagery isn't available yet (future/pre-season dates), estimates peak vegetation index from a multi-year trajectory formula:

  ```
  NDVI(t) = 0.60·NDVI(t-1) + 0.40·mean_3yr + 0.50·(NDVI(t-1) - NDVI(t-2))
  ```

### Two-stage prediction
1. **Stage 1 — Baseline yield (Y₀):** an ExtraTrees / XGBoost regression model predicts environmental yield from district soil physics, historical meteorology, and remote sensing.
2. **Stage 2 — Fertilizer uplift (ΔY):** the fertilizer engine decomposes the applied product (e.g. "150 kg Urea + 100 kg DAP") into exact elemental nutrient doses and applies a quadratic response curve with a nitrogen-phosphorus synergy term to compute the yield gain.

Models were trained on 2016/17–2023 data and tested on a held-out 2024 season, with results ranging from **R² = 0.89 on Wheat** down to **R² = 0.52 on Rice**, depending on crop.

---

## 3. Repository Structure

```
DigitalTwin/
├── Assets/                    # Unity project — the digital twin visualization
├── Packages/                  # Unity package manifest
├── ProjectSettings/           # Unity project settings
├── .vscode/                   # Editor config
│
├── main.py                        # Baseline single global XGBoost model (all crops)
├── 6_cropy.py                     # 6 core crops — ExtraTrees training pipeline
├── major_crops.py                 # Major crops — XGBoost training pipeline
├── minor_crops.py                 # Minor crops — ExtraTrees training pipeline
├── train_ex_ante_models.py        # 45-crop production "decision-time" model trainer
├── new_algo_test.py               # Optuna hyperparameter search (LightGBM/CatBoost/ExtraTrees)
│
├── yield_prediction_backend.py    # Master Google Earth Engine data fetcher
├── gee_live_fetcher.py            # On-demand satellite fetch + local cache
├── build_ex_ante_features.py      # Builds lagged, no-lookahead-bias decision-time features
├── ndvi_fallback_engine.py        # Multi-year NDVI trajectory estimator (pre-season fallback)
│
├── predict.py                     # Main prediction engine — CLI + importable function
├── fertilizer_engine.py           # Nutrient decomposition + quadratic response curve
├── api_server.py                  # Lightweight CORS-enabled REST API (stdlib only)
├── evaluate.py                    # Model benchmarking suite (MAE, RMSE, MAPE, R²)
│
├── ex_ante_crop_models/           # 45 production models (one per crop)
├── six_crop_extratrees_models/    # 6 core crop models
├── major_crop_models/             # Major crop XGBoost models
├── remaining_crop_extratrees/     # Minor crop / vegetable models
│
├── punjab_model1_master_with_s1_weather_soil.csv   # Master historical training dataset
├── forecast_feature_cache_2025_2030.csv            # Cached future climatology features
├── requirements.txt
└── setup_env.ps1                  # Windows environment/venv setup script
```

---

## 4. Setup

### Prerequisites
- Python 3.10+
- Unity (version matching `ProjectSettings/`) for the front-end
- A [Google Earth Engine](https://earthengine.google.com/) account (only needed if you want to pull fresh satellite data — cached CSVs are included for offline use)

### Install (Python backend)
```bash
git clone https://github.com/dureadaaan/DigitalTwin.git
cd DigitalTwin
python -m venv .venv
.venv\Scripts\activate          # Windows
pip install -r requirements.txt
```

On Windows, `setup_env.ps1` will additionally create the venv and redirect pip/temp caches to avoid filling up the `C:` drive:
```powershell
.\setup_env.ps1
```

### Run a prediction from the CLI
```bash
python predict.py --crop Rice --district Gujranwala --sowing_date "06/15/2026" --fertilizer "High Nitrogen (Urea + DAP)"
```

Add `--json` to get machine-readable output for a front-end:
```bash
python predict.py --crop Wheat --district Faisalabad --fertilizer "Balanced NPK (15-15-15)" --json
```

### Run the REST API (for the Unity front-end / any web client)
```bash
python api_server.py --port 8000
```

Then POST to it:
```bash
curl -X POST http://localhost:8000/predict \
  -H "Content-Type: application/json" \
  -d '{"crop": "Rice", "district": "Gujranwala", "sowing_date": "06/15/2026", "fertilizer": "High Nitrogen (Urea + DAP)"}'
```

CORS is enabled for all origins, so `localhost:3000`, `localhost:5173`, and the Unity WebGL build can all call it directly.

### Open the Unity project
Open the repository root in Unity Hub (it will detect `Assets/`, `Packages/`, and `ProjectSettings/`) and point the front-end's API calls at wherever `api_server.py` is running.

### Evaluate model performance
```bash
python evaluate.py --years 2024 --top 10
```

---

## 5. Sample Results

| Crop | R² (2024 holdout) | MAPE |
|---|---|---|
| Wheat | 0.89 | 6.9% |
| Jowar | 0.84 | 21.3% |
| Gram | 0.79 | 66.6% |
| Kharif Fodder | 0.79 | 7.2% |
| Rapeseed & Mustard | 0.71 | 10.8% |
| Sugarcane | 0.65 | 7.9% |
| Rice | 0.52 | 8.6% |

Example — Rice, Gujranwala, sown 06/15/2026:
- No fertilizer: **~22.5 maunds/acre**
- + High Nitrogen (Urea + DAP): **~27.3 maunds/acre** (+4.8)
- + Zinc Fortified Blend: **~26.5 maunds/acre** (+4.0)

---

## 6. Team / Credits

Built during a hackathon on digital twins in agriculture. Backend ML/remote-sensing pipeline and Unity rendering/simulation integration by the team.

---

## 7. Notes & Limitations

- Some crop models (e.g. Cucumber, Mango, Gram) have low sample counts and high MAPE — treat their outputs as directional, not exact.
- `punjab_model1_master_with_s1_weather_soil.csv` and the `.pkl` model files are included so the app runs fully offline without a GEE account.
- `.env` and `setup_env.ps1` assume a Windows dev environment with data cached off the `C:` drive; adjust paths for other OS setups.
