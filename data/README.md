# Crop datasets

Two CSV files for training a crop yield / profit estimation model. Data is
synthetically generated (not real agricultural records) but follows
plausible ranges and trends so it can be used to prototype a model.

## `crop_yield.csv`

One row per Crop + Region + Year (240 rows, 8 crops x 5 regions x 6 years, 2019-2024).

| Column | Description |
|---|---|
| `Crop` | Crop name (Wheat, Rice, Maize, Soybean, Cotton, Sugarcane, Barley, Potato) |
| `Region` | North / South / East / West / Central |
| `Season` | Kharif or Rabi |
| `Year` | 2019-2024 |
| `Area_hectares` | Cultivated area |
| `Rainfall_mm` | Seasonal rainfall |
| `Avg_Temperature_C` | Average seasonal temperature |
| `Fertilizer_kg_per_hectare` | Fertilizer application rate |
| `Cost_of_Cultivation_per_hectare_USD` | Input + labor cost per hectare |
| `Yield_tonnes_per_hectare` | Target variable for yield estimation |

## `market_price.csv`

One row per Crop + Year + Month (576 rows, 8 crops x 6 years x 12 months).

| Column | Description |
|---|---|
| `Crop` | Crop name (matches `crop_yield.csv`) |
| `Year` | 2019-2024 |
| `Month` | 1-12 |
| `Market_Price_per_tonne_USD` | Market price for that crop/month |

## Suggested use

1. Train a yield model on `crop_yield.csv` (features: Region, Season, Rainfall,
   Temperature, Fertilizer, etc. -> target: `Yield_tonnes_per_hectare`).
2. Join the predicted yield with `market_price.csv` on `Crop` + `Year` (and
   `Month`, if estimating for a specific month) to get an expected price.
3. Estimate profit per hectare as:

   ```
   profit_per_ha = predicted_yield_t_per_ha * market_price_per_tonne
                   - cost_of_cultivation_per_ha
   ```

   Multiply by `Area_hectares` for total farm profit.
