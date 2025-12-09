# Haier Europe 2025 Datathon - Demand Forecasting Solution

This repository contains the solution for the Haier Europe 2025 Datathon. The objective was to generate a robust 12-month demand forecast for over 50,000 SKUs, ensuring consistency between SKU-level and Product Line (Category) forecasts while correctly handling product life-cycles.

**Final Rank:** 10th Place 🏆

## 📊 The Metric: rWMAPE
The competition used **Regularized Weighted MAPE (rWMAPE)**.

## 🛠️ Solution Architecture

### 1. Modeling with AutoGluon TimeSeries
We utilized **AutoGluon TimeSeries** as the core engine.
*   **Preset:** `high_quality` (Ensembles DeepAR, TiDE, PatchTST, and Gradient Boosting).
*   **Training Data:** We "densified" the sparse transaction data.
    *   **Interpolation:** Used linear interpolation to fill gaps in intermediate months. This smoothed the variance and helped Tree/DL models learn trends better than zero-filling, minimizing the Volume Penalty.
*   **Features:** Engineered lifecycle features derived from `product_master` (e.g., `months_since_start_production`, `months_since_end_production`).

### 2. Forecast Aggregation (Consistency)
To satisfy the requirement that SKU forecasts must match Line-level forecasts, we used a **Bottom-Up Aggregation** strategy:
1.  Predict at the **SKU Level** (Market-Product).
2.  Apply post-processing and clipping.
3.  **Sum** the valid SKU forecasts to generate the **Category Level** forecasts.
*   *Result:* Guaranteed mathematical consistency and avoided "hallucinating" sales for products not present in specific markets.

## ✅ What Worked
*   **Interpolation for Training:** Training on interpolated data (filling gaps) significantly improved the score by forcing the model to predict volume.
*   **Continuous Lifecycle Features:** Using `months_since_endprod` (continuous) worked better than a binary `is_active` flag, allowing models to learn the natural decay curve.
*   **Bottom-Up Aggregation:** Aggregating cleaned SKU predictions was superior to predicting Categories directly.
*   **Global Models:** Deep Learning models (TiDE/DeepAR via AutoGluon) outperformed local statistical models on the 50k+ series dataset.

## ❌ What Did Not Work
*   **Binary Phase-Out Flags:** Feeding a hard `0/1` active flag during training confused tree models at the boundary. Continuous features were superior.
*   **Standard Validation:** Validating on sparse data gave misleadingly high scores.
*   **Prophet:** Too slow and memory-intensive for 50,000 global series.
*   **Darts and Nixtla Libraries:**: Data is too sparse for these libraries.
*   **Direct Category Prediction:** Predicting at the category level directly caused volume mismatches and included discontinued products in the totals.
*   **Phase-Out Handling (The "Kill Switch"):** I analyzed the training data and found that 75% of products stop selling within 2 years of EOP. Applied a hard cutoff. If the forecast date was > `end_production_date` + 730 days (Buffer), the prediction was forced to **0**.
*   **Clipping negatives to zero in target** 
