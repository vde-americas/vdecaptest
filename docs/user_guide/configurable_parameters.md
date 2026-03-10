# Configurable Parameters Documentation

## Overview

This document lists all user-configurable parameters in pvcaptest that should be reviewed and adjusted for each new capacity test analysis.

## 1. Standard Selection

**Parameter:** `standard`

- **Location:** Set when creating CapData object or loading data
- **Options:** `'ASTM'` (default) or `'IEC'`
- **How to set:**
  ```python
  # When loading data
  das = ct.load_data('path/to/data.csv', standard='ASTM')  # or 'IEC'
  sim = ct.load_pvsyst('path/to/pvsyst.csv', standard='ASTM')
  
  # When creating CapData directly
  cd = CapData('name', standard='ASTM')
  ```

- **When to change:** Select based on which standard you're following (ASTM E2848 vs IEC 61724-2)
- **Impact:** Changes regression formula and required columns

## 2. Module Type Configuration

### 2.1 Monofacial vs Bifacial

**For Bifacial Modules:**

- **Bifaciality Factor:** User must calculate `E_Total = E_POA + E_Rear * bifaciality`
- **How to set:**
  ```python
  # Calculate total irradiance
  bifaciality = 0.7  # Example: 70% bifaciality factor
  das.data['E_Total'] = das.data['E_POA'] + das.data['E_Rear'] * bifaciality
  das.data_filtered = das.data.copy()
  
  # Update regression columns to use E_Total instead of E_POA
  das.set_regression_cols(power='power_col', poa='E_Total', t_amb='temp_col', w_vel='wind_col')
  ```

- **When to change:** For bifacial module projects
- **Reference:** See `docs/user_guide/bifacial.rst`

**For Monofacial Modules:**

- Use standard POA irradiance (no changes needed)

### 2.2 IEC Module Type Parameters

**Parameter:** `module_type` and `racking` (for IEC standard only)

- **Location:** `CapData.set_iec_params()`
- **Options:**
  - `module_type`: `'glass_cell_poly'` (default), `'glass_cell_glass'`, `'poly_tf_steel'`
  - `racking`: `'open_rack'` (default), `'close_roof_mount'`, `'insulated_back'`
- **How to set:**
  ```python
  das.set_iec_params(
      module_type='glass_cell_poly',  # or 'glass_cell_glass', 'poly_tf_steel'
      racking='open_rack'  # or 'close_roof_mount', 'insulated_back'
  )
  ```

- **When to change:** For IEC standard, based on actual module and racking configuration
- **Impact:** Affects cell temperature calculation for temperature correction

## 3. Data Loading Parameters

**Parameters:**

- `path`: Path to data file(s) or directory
- `name`: Identifier for CapData object (default: `'meas'`)
- `group_columns`: Column grouping function or file path
- `standard`: `'ASTM'` or `'IEC'` (default: `'ASTM'`)
- `site`: Dictionary or file path with location/system info for clear sky modeling

**How to set:**

```python
# Basic loading
das = ct.load_data('path/to/data.csv', name='measured_data', standard='ASTM')

# With custom column groups
das = ct.load_data('path/to/data.csv', group_columns='path/to/column_groups.xlsx')

# With site info for clear sky modeling
site = {
    'loc': {'latitude': 40.0, 'longitude': -105.0, 'altitude': 1600, 'tz': 'America/Denver'},
    'sys': {'surface_tilt': 20, 'surface_azimuth': 180, 'albedo': 0.2}
}
das = ct.load_data('path/to/data.csv', site=site)
```

**When to change:** Every new project - update file paths and site information

## 4. Column Grouping

**Parameter:** `column_groups`

- **Location:** Automatically created, but can be manually set or loaded from file
- **Options:** Dictionary, Excel file, JSON file, or YAML file
- **How to set:**
  ```python
  # Review automatic grouping
  das.column_groups
  
  # Load from Excel (recommended workflow)
  # 1. Generate template
  das = ct.load_data('data.csv', column_groups_template=True)
  # 2. Edit the generated Excel file
  # 3. Reload with custom grouping
  das = ct.load_data('data.csv', group_columns='column_groups.xlsx')
  
  # Or set manually
  das.column_groups = {
      'irr_poa': ['met1_poa', 'met2_poa'],
      'real_pwr': ['meter_power'],
      'temp_amb': ['amb_temp']
  }
  ```

- **When to change:** When automatic grouping is incorrect (common for different DAS systems)

## 5. Regression Column Mapping

**Parameter:** `regression_cols`

- **Location:** `CapData.set_regression_cols()`
- **Required mappings:** `power`, `poa`, `t_amb`, `w_vel` (for ASTM); `power`, `poa` (for IEC)
- **How to set:**
  ```python
  # Using column group keys
  das.set_regression_cols(
      power='real_pwr_mtr',
      poa='irr_poa',
      t_amb='temp_amb',
      w_vel='wind'
  )
  
  # Using direct column names
  das.set_regression_cols(
      power='Meter_Power',
      poa='POA_Irradiance',
      t_amb='Ambient_Temp',
      w_vel='Wind_Speed'
  )
  ```

- **When to change:** Every new project - column names vary by data source
- **Impact:** Critical - incorrect mapping will cause regression to fail

## 6. Sensor Aggregation

**Parameter:** `agg_map` in `agg_sensors()`

- **Location:** `CapData.agg_sensors()`
- **Default:** Sums power, averages poa/t_amb/w_vel
- **How to set:**
  ```python
  # Use defaults (sum power, mean others)
  das.agg_sensors()
  
  # Custom aggregation
  das.agg_sensors(agg_map={
      'real_pwr_inv': 'sum',      # Sum inverter power
      'irr_poa': 'mean',          # Average POA irradiance
      'temp_amb': 'mean',         # Average ambient temp
      'wind': 'mean'              # Average wind speed
  })
  ```

- **When to change:** When multiple sensors exist and default aggregation is inappropriate

## 7. Filtering Parameters

### 7.1 Irradiance Filter

**Parameter:** `filter_irr(low, high, ref_val, col_name)`

- **Options:**
  - `low`: Minimum irradiance (absolute W/m² or fraction like 0.8)
  - `high`: Maximum irradiance (absolute W/m² or fraction like 1.2)
  - `ref_val`: Reference value when using fractions (or `'self_val'` to use reporting conditions)
  - `col_name`: Specific column name (optional)
- **How to set:**
  ```python
  # Absolute values (W/m²)
  das.filter_irr(200, 1200)
  
  # Fractional values around reference
  das.filter_irr(0.8, 1.2, ref_val=800)  # 80% to 120% of 800 W/m²
  
  # Using reporting conditions (after rep_cond is calculated)
  das.filter_irr(0.5, 1.5, ref_val='self_val')
  ```

- **When to change:** Adjust based on data quality and test requirements
- **Typical values:** Initial filter: 200-1200 W/m², Final filter: ±50% around reporting irradiance

### 7.2 Sensor Consistency Filter

**Parameter:** `filter_sensors(perc_diff)`

- **Options:**
  - `perc_diff`: Dictionary mapping sensor groups to percent difference thresholds (default: 5% for POA)
- **How to set:**
  ```python
  # Default (5% for POA sensors)
  das.filter_sensors()
  
  # Custom thresholds
  das.filter_sensors(perc_diff={'irr_poa': 0.05, 'temp_amb': 0.10})
  ```

- **When to change:** Adjust threshold based on sensor accuracy and data quality

### 7.3 Outlier Filter

**Parameter:** `filter_outliers(**kwargs)`

- **Options:**
  - `contamination`: Proportion of outliers (default: 0.04 = 4%)
  - `support_fraction`: Support fraction (default: 0.9)
- **How to set:**
  ```python
  # Default (4% contamination)
  das.filter_outliers()
  
  # Custom contamination
  das.filter_outliers(contamination=0.05)  # 5% outliers
  ```

- **When to change:** Adjust based on data quality

### 7.4 Power Filter

**Parameter:** `filter_power(power, percent, columns)`

- **Options:**
  - `power`: Absolute power threshold or nameplate (if using percent)
  - `percent`: Percentage of nameplate (as decimal, e.g., 0.01 for 1%)
  - `columns`: Column or column group to filter on
- **How to set:**
  ```python
  # Absolute threshold
  das.filter_power(6000000)  # Remove data >= 6 MW
  
  # Percentage of nameplate
  das.filter_power(6000000, percent=0.01)  # Remove data >= 99% of 6 MW nameplate
  ```

- **When to change:** Remove clipping or over-power conditions

### 7.5 Time Filter

**Parameter:** `filter_time(start, end, days, test_date, wrap_year)`

- **Options:**
  - `start`: Start date (string or Timestamp)
  - `end`: End date (string or Timestamp)
  - `days`: Number of days
  - `test_date`: Center date for period
  - `wrap_year`: Boolean for year-end wrapping
- **How to set:**
  ```python
  # Date range
  das.filter_time(start='2023-01-01', end='2023-12-31')
  
  # Days from start
  das.filter_time(start='2023-06-01', days=60)
  
  # Centered on test date
  das.filter_time(test_date='2023-07-15', days=60)
  ```

- **When to change:** Select test period for analysis

### 7.6 Clear Sky Filter

**Parameter:** `filter_clearsky(window_length, ghi_col, keep_clear)`

- **Options:**
  - `window_length`: Sliding window in minutes (default: 20 for 5-min data, 10 for 1-min)
  - `ghi_col`: Column name for measured GHI (auto-detected if None)
  - `keep_clear`: Boolean - True keeps clear periods, False keeps cloudy
- **How to set:**
  ```python
  # Default (keep clear periods)
  das.filter_clearsky()
  
  # Custom window length
  das.filter_clearsky(window_length=30)  # 30-minute window
  
  # Keep cloudy periods instead
  das.filter_clearsky(keep_clear=False)
  ```

- **When to change:** Adjust window length based on data interval

### 7.7 Shade Filter

**Parameter:** `filter_shade(fshdbm, query_str)`

- **Options:**
  - `fshdbm`: Fractional shading threshold (default: 1.0 = no shading)
  - `query_str`: Custom query string for shading data
- **How to set:**
  ```python
  # Default (remove all shading)
  das.filter_shade()
  
  # Allow some shading
  das.filter_shade(fshdbm=0.95)  # Remove if shading > 5%
  
  # Custom query
  das.filter_shade(query_str='ShdLoss<=50')  # Remove if shading loss > 50 W
  ```

- **When to change:** For PVsyst data or when shading data available

### 7.8 Power Factor Filter

**Parameter:** `filter_pf(pf)`

- **Options:**
  - `pf`: Minimum power factor (default: 0.999)
- **How to set:**
  ```python
  das.filter_pf(0.999)  # Keep only data with PF >= 0.999
  ```

- **When to change:** When power factor data available and quality issues exist

### 7.9 Missing Data Filter

**Parameter:** `filter_missing(columns)`

- **Options:**
  - `columns`: List of columns to check (default: regression columns)
- **How to set:**
  ```python
  # Default (check regression columns)
  das.filter_missing()
  
  # Specific columns
  das.filter_missing(columns=['power', 'poa', 't_amb'])
  ```

- **When to change:** Remove periods with missing critical data

## 8. Reporting Conditions Parameters

**Parameter:** `rep_cond(irr_bal, percent_filter, func, freq, w_vel, rc_kwargs)`

- **Options:**
  - `irr_bal`: Boolean - use balanced irradiance method (default: False)
  - `percent_filter`: Percent band for balanced method (default: 20%)
  - `func`: Aggregation function - dict, string, or callable (default: 60th percentile for POA, mean for others)
  - `freq`: Frequency for grouping (e.g., `'MS'` for monthly, `'60D'` for 60-day periods)
  - `w_vel`: Override wind speed reporting condition
  - `rc_kwargs`: Additional kwargs for ReportingIrradiance class
- **How to set:**
  ```python
  # Default (60th percentile POA, mean temp/wind)
  das.rep_cond()
  
  # Balanced irradiance method
  das.rep_cond(irr_bal=True, percent_filter=20)
  
  # Monthly reporting conditions
  das.rep_cond(freq='MS', irr_bal=True, percent_filter=20)
  
  # Custom aggregation
  das.rep_cond(func={'poa': 'median', 't_amb': 'mean', 'w_vel': 'mean'})
  
  # With ReportingIrradiance parameters
  das.rep_cond(
      irr_bal=True,
      percent_filter=20,
      rc_kwargs={
          'min_percent_below': 40,
          'max_percent_above': 60,
          'min_ref_irradiance': 600,
          'max_ref_irradiance': 900,
          'points_required': 750
      }
  )
  ```

- **When to change:** 
  - `irr_bal=True` recommended for better data balance
  - `freq` for seasonal/monthly analysis
  - `rc_kwargs` to adjust reporting irradiance selection criteria

### 8.1 ReportingIrradiance Advanced Parameters

**Parameters:** (passed via `rc_kwargs` in `rep_cond()`)

- `percent_band`: Percent band around reporting irradiance (default: 20, range: 2-50)
- `min_percent_below`: Minimum % points below RC (default: 40)
- `max_percent_above`: Maximum % points above RC (default: 60)
- `min_ref_irradiance`: Minimum allowed reference irradiance (default: auto-calculated)
- `max_ref_irradiance`: Maximum allowed reference irradiance (default: auto-calculated)
- `points_required`: Minimum points required (default: 750)

## 9. IEC-Specific Parameters

**Parameter:** `set_iec_params(beta, delta_t, e_ref, t_stc, module_type, racking)`

- **Options:**
  - `beta`: Temperature coefficient (%/°C) - **REQUIRED for IEC**
  - `delta_t`: Temperature difference constant (°C, default: 3.0)
  - `e_ref`: Reference irradiance (W/m², default: 1000.0)
  - `t_stc`: STC temperature (°C, default: 25.0)
  - `module_type`: See section 2.2
  - `racking`: See section 2.2
- **How to set:**
  ```python
  das.set_iec_params(
      beta=-0.35,           # -0.35%/°C (typical for silicon modules)
      delta_t=3.0,          # Default for flat-plate
      e_ref=1000.0,         # Standard reference
      t_stc=25.0,           # Standard test conditions
      module_type='glass_cell_poly',
      racking='open_rack'
  )
  ```

- **When to change:** 
  - `beta` must be set from module datasheet
  - `delta_t` may vary by module type (typically 3.0 for flat-plate)
  - Other parameters usually use defaults

## 10. Regression Parameters

**Parameter:** `fit_regression(filter, inplace, summary)`

- **Options:**
  - `filter`: Boolean - filter outliers using residuals (default: False)
  - `inplace`: Boolean - modify data_filtered (default: True)
  - `summary`: Boolean - print regression summary (default: True)
- **How to set:**
  ```python
  # Standard regression
  das.fit_regression()
  
  # With residual filtering (removes points > 2 std dev)
  das.fit_regression(filter=True)
  
  # Silent (no summary printed)
  das.fit_regression(summary=False)
  ```

- **When to change:** Use `filter=True` if outliers remain after other filtering

## 11. Capacity Test Results Parameters

**Parameter:** `captest_results(sim, das, nameplate, tolerance, check_pvalues, pval, print_res)`

- **Options:**
  - `nameplate`: Nameplate capacity (numeric)
  - `tolerance`: Error band string (e.g., `'+/- 7'`, `'- 5'`)
  - `check_pvalues`: Boolean - zero out high p-value coefficients (default: False)
  - `pval`: P-value threshold (default: 0.05)
  - `print_res`: Boolean - print results (default: True)
- **How to set:**
  ```python
  # Basic results
  ct.captest_results(sim, das, nameplate=6000000, tolerance='+/- 7')
  
  # With p-value checking
  ct.captest_results(
      sim, das,
      nameplate=6000000,
      tolerance='+/- 7',
      check_pvalues=True,
      pval=0.05
  )
  
  # Or use the check_pvalues version
  ct.captest_results_check_pvalues(
      sim, das,
      nameplate=6000000,
      tolerance='+/- 7',
      print_res=True
  )
  ```

- **When to change:** 
  - `nameplate`: Project-specific
  - `tolerance`: Contract or standard requirement
  - `check_pvalues=True`: Recommended to check coefficient significance

## 12. Points Required

**Parameter:** `get_pts_required(hrs_req)`

- **Options:**
  - `hrs_req`: Hours of data required (default: 12.5 hours = 750 points for 1-min data)
- **How to set:**
  ```python
  das.get_pts_required(hrs_req=12.5)  # ASTM E2848 default
  das.print_points_summary(hrs_req=12.5)
  ```

- **When to change:** If standard requires different minimum hours

## 13. Spectral Correction Parameters

**Parameter:** `set_spectral_params(enabled, module_type, airmass, precipitable_water, location)`

Spectral correction adjusts POA irradiance for spectral mismatch effects, which is important for thin-film modules (especially First Solar CdTe modules) where the spectral response differs from crystalline silicon reference cells.

- **Options:**
  - `enabled`: Boolean - whether to enable spectral correction (default: False)
  - `module_type`: str - First Solar module type. Options: `'cdte'`, `'monosi'`, `'multisi'`, `'polysi'`, `'cigs'`, `'asi'`. Required if enabled is True.
  - `airmass`: float or Series, optional - Air mass values. If not provided, will be calculated from location and timestamps.
  - `precipitable_water`: float or Series, optional - Precipitable water in cm. If not provided, will be estimated from location or use default (1.0 cm).
  - `location`: dict, optional - Location dictionary with `'latitude'`, `'longitude'`, `'altitude'`, `'tz'`. Required if airmass or precipitable_water need to be calculated.

- **How to set:**
  ```python
  # Basic setup with module type and location (auto-calculates airmass/precipitable_water)
  das.set_spectral_params(
      enabled=True,
      module_type='cdte',  # For First Solar CdTe modules
      location={
          'latitude': 40.0,
          'longitude': -105.0,
          'altitude': 1600,
          'tz': 'America/Denver'
      }
  )
  
  # With all parameters provided
  das.set_spectral_params(
      enabled=True,
      module_type='cdte',
      airmass=1.5,  # Single value or Series
      precipitable_water=1.2,  # Single value or Series in cm
      location={'latitude': 40.0, 'longitude': -105.0, 'altitude': 1600, 'tz': 'America/Denver'}
  )
  
  # With time-varying airmass and precipitable water
  airmass_series = pd.Series([1.2, 1.5, 2.0], index=das.data_filtered.index[:3])
  pw_series = pd.Series([0.8, 1.0, 1.2], index=das.data_filtered.index[:3])
  das.set_spectral_params(
      enabled=True,
      module_type='cdte',
      airmass=airmass_series,
      precipitable_water=pw_series
  )
  ```

- **When to change:**
  - Enable for thin-film modules (CdTe, CIGS, a-Si) where spectral mismatch is significant
  - Required for First Solar modules when spectral correction is specified by test methodology
  - Typically not needed for crystalline silicon modules

- **Impact:**
  - Automatically applies spectral correction to POA irradiance before regression
  - Corrected POA is used in all regression calculations (ASTM and IEC standards)
  - Reporting conditions also use spectrally-corrected POA values
  - Corrected POA is stored in `data_filtered` as `poa_spectral_corrected` for reference

- **Workflow Integration:**
  - Spectral correction is applied automatically in `fit_regression()` if enabled
  - Reporting conditions calculated by `rep_cond()` use corrected POA if enabled
  - Predictions use corrected reporting conditions if spectral correction was used in regression

## Summary Checklist for Each New Capacity Test

1. **Standard:** Set `standard='ASTM'` or `'IEC'` when loading data
2. **Module Type:** 
   - For bifacial: Calculate E_Total and update regression columns
   - For IEC: Set `beta` and optionally `module_type`/`racking`
3. **Data Paths:** Update file paths in `load_data()` or `load_pvsyst()`
4. **Column Groups:** Review and correct automatic grouping (use Excel template if needed)
5. **Regression Columns:** Map power, poa, t_amb, w_vel to actual column names/groups
6. **Sensor Aggregation:** Run `agg_sensors()` if multiple sensors exist
7. **Filtering:** Adjust filter thresholds based on data quality:
   - Initial irradiance filter (typically 200-1200 W/m²)
   - Sensor consistency (typically 5%)
   - Outlier removal (typically 4% contamination)
   - Final irradiance filter around reporting conditions (±50%)
8. **Reporting Conditions:** 
   - Use `irr_bal=True` with `percent_filter=20` (recommended)
   - Adjust `rc_kwargs` if needed for reporting irradiance selection
9. **Nameplate & Tolerance:** Set in `captest_results()` call
10. **P-value Check:** Use `check_pvalues=True` to verify coefficient significance
11. **Spectral Correction:** If using thin-film modules, enable and configure spectral correction before regression