# Capacity Test Template

This template provides a starting point for conducting capacity tests using pvcaptest following ASTM E2848 or IEC 61724-2 standards.

## Quick Start

1. **Copy this template** to a new project directory:
   ```bash
   cp -r projects/_template projects/my_project_name
   cd projects/my_project_name
   ```

2. **Update configuration files:**
   - Edit `site.yml` with your project location and system parameters
   - Update `plot_defaults.json` if needed (optional)

3. **Prepare your data:**
   - Replace `data/example_measured_data.csv` with your measured data
   - Replace `pvsyst/pvsyst_output_template.csv` with your PVsyst simulation output
   - Generate and edit `column_groups.xlsx` if automatic column grouping is incorrect

4. **Open and customize the notebook:**
   - Open `captest_template.ipynb` in Jupyter Lab
   - Follow the notebook sections, updating all parameters marked with `[UPDATE]` or placeholders
   - Refer to `docs/user_guide/configurable_parameters.md` for detailed parameter explanations

5. **Run the analysis:**
   - Execute cells in order
   - Review and adjust filter parameters as needed
   - Verify results meet test requirements

## Directory Structure

```
_template/
├── README.md                      # This file
├── captest_template.ipynb         # Main analysis notebook template
├── site.yml                       # Site location and system configuration
├── plot_defaults.json             # Plot configuration (optional)
├── environment.yml                # Conda environment specification
├── column_groups.xlsx             # Column grouping definition (generate if needed)
├── data/                          # Measured data directory
│   ├── example_measured_data.csv  # Template CSV (replace with your data)
│   └── column_groups.xlsx         # Column groups (if generated)
├── pvsyst/                        # PVsyst simulation data directory
│   └── pvsyst_output_template.csv # Template CSV (replace with your PVsyst output)
└── images/                        # Project images/logos (optional)
```

## File Descriptions

### Configuration Files

- **`site.yml`**: Site location (latitude, longitude, altitude, timezone) and system parameters (tilt, azimuth, albedo). Required for clear sky modeling.
- **`plot_defaults.json`**: Default column groups for plotting. Optional, can be customized.
- **`environment.yml`**: Conda environment with all required packages. Use `conda env create -f environment.yml` to create the environment.

### Data Files

- **`data/example_measured_data.csv`**: Template for measured SCADA/DAS data. Replace with your actual data file. Should include:
  - Timestamp column (will become DataFrame index)
  - POA irradiance measurements
  - Power measurements (meter and/or inverter level)
  - Ambient temperature
  - Wind speed
  - Optional: Rear POA irradiance for bifacial systems

- **`pvsyst/pvsyst_output_template.csv`**: Template for PVsyst hourly output. Replace with your actual PVsyst export. Should include columns like `GlobInc`, `T_Amb`, `WindVel`, `E_Grid`, etc.

- **`column_groups.xlsx`**: Excel file defining column groupings. Generated automatically if `column_groups_template=True` in `load_data()`. Edit this file if automatic grouping is incorrect.

### Notebook

- **`captest_template.ipynb`**: Comprehensive template notebook covering all 12 categories of configurable parameters:
  1. Standard selection (ASTM/IEC)
  2. Module type configuration (monofacial/bifacial, IEC params)
  3. Data loading parameters
  4. Column grouping
  5. Regression column mapping
  6. Sensor aggregation
  7. All filtering parameters (9 different filter types)
  8. Reporting conditions parameters
  9. IEC-specific parameters
  10. Regression parameters
  11. Capacity test results parameters
  12. Points required verification

## Configurable Parameters Checklist

**IMPORTANT:** Review and customize all parameters in the notebook. Refer to `docs/user_guide/configurable_parameters.md` for detailed explanations.

### Required for Every Project

- [ ] **Standard**: Select ASTM or IEC
- [ ] **Data paths**: Update paths to your data files
- [ ] **Site configuration**: Update `site.yml` with location and system parameters
- [ ] **Column groups**: Review automatic grouping, create custom if needed
- [ ] **Regression columns**: Map power, poa, t_amb, w_vel to your column names
- [ ] **Nameplate**: Set AC nameplate capacity
- [ ] **Tolerance**: Set capacity test tolerance

### Project-Specific (as applicable)

- [ ] **Bifacial systems**: Set bifaciality factor and calculate E_Total
- [ ] **IEC standard**: Set beta (temperature coefficient) and module/racking type
- [ ] **Sensor aggregation**: Configure if multiple sensors exist
- [ ] **Filter parameters**: Adjust filter thresholds based on data quality
- [ ] **Reporting conditions**: Customize method and parameters
- [ ] **P-value checking**: Enable to verify coefficient significance

## Workflow Overview

1. **Setup**: Import packages, set global parameters
2. **Standard Selection**: Choose ASTM or IEC
3. **Data Loading**: Load measured and PVsyst data
4. **Column Grouping**: Review and correct automatic grouping
5. **Regression Mapping**: Map regression variables to data columns
6. **Filtering**: Apply filters in logical sequence
7. **Reporting Conditions**: Calculate reporting conditions
8. **Regression**: Fit regression models
9. **Results**: Calculate capacity ratio and pass/fail
10. **Verification**: Check data sufficiency (points required)

## Resources

- **pvcaptest Documentation**: https://pvcaptest.readthedocs.io/en/latest/
- **Configurable Parameters Guide**: `docs/user_guide/configurable_parameters.md` - **Essential reading**
- **Source Code**: https://github.com/pvcaptest/pvcaptest
- **ASTM E2848**: https://www.astm.org/e2848-13r18.html
- **ASTM E2939**: https://www.astm.org/e2939-13r18.html

## Tips

- **Start with defaults**: Many parameters have sensible defaults. Start with defaults and adjust as needed.
- **Filter order matters**: Apply filters in logical sequence. Initial irradiance filter → sensor consistency → outliers → clear sky → final irradiance filter.
- **Use balanced irradiance**: Recommended to use `irr_bal=True` in `rep_cond()` for better data balance.
- **Check p-values**: Use `captest_results_check_pvalues()` to verify regression coefficient significance.
- **Visualize data**: Use plotting functions (`plot()`, `scatter_hv()`, `scatter_filters()`) to explore data quality.
- **Document filtering**: Export filtering documentation for reporting and review.

## Troubleshooting

- **Column grouping incorrect**: Generate template with `column_groups_template=True`, edit in Excel, reload data.
- **Regression fails**: Verify regression columns are correctly mapped and data is present.
- **Insufficient data**: Check `get_pts_required()` output. May need to adjust filter parameters or extend test period.
- **Clear sky filter too strict**: Adjust `window_length` parameter or disable if not required by contract.

## Support

For issues, questions, or contributions:
- GitHub Issues: https://github.com/pvcaptest/pvcaptest/issues
- Documentation: https://pvcaptest.readthedocs.io/
