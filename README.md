# Bioprocess Analytics Dashboard with PowerBI

### Project Overview
This repository showcases an end-to-end **Bioprocess Monitoring** built in **Microsoft Power BI**. 
Developed as my first-ever hands-on project in Power BI, this project translates complex upstream Chinese Hamster Ovary (CHO) cell cultivation data into a dashboard visualization for process insights. 

---

### Dataset
**Source:** Dataset Based on Chinese Hamster Ovary (CHO) Cultivations including Turbidity, Permittivity, O2 and CO2 Measurements [(Zenodo: 10.5281/zenodo.20829178)](https://doi.org/10.5281/zenodo.20829178)

**Scope:** 24 bioreactor cultivation runs tracking both daily offline laboratory samples and continuous high-frequency inline sensor telemetry.

> **Citation:**
>
> Uhlendorff, S., Fulek, R., Eimler, J., Pein-Hackelbusch, M., & Frahm, B. (2026). Dataset Based on Chinese Hamster Ovary (CHO) Cultivations including Turbidity, Permittivity, O2 and CO2 Measurements (Version Version v1) [Dataset]. Zenodo. https://doi.org/10.5281/zenodo.20829178

---

### Data Architecture: 
```text
                      +-----------------------------+
                      |       Dim_Experiment        |
                      |-----------------------------|
                      |     * Experiment_ID         |
                      +--------------+--------------+
                                     |
                   +-----------------+-----------------+
                   | 1:Many (Single)                   | 1:Many (Single)
                   v                                   v
     +---------------------------+       +---------------------------+
     |        Offline_Data       |       |        Inline_Data        |
     |---------------------------|       |---------------------------|
     | * Experiment_ID           |       | * Experiment_ID           |
     | - Time (h)                |       | - Time (h)                |
     | - VCD Mean (cells/mL)     |       | - pH                      |
     | - Viability Mean (%)      |       | - Dissolved O2 (% air sat)|
     | - Glucose (g/L)           |       | - Dissolved CO2 (%-sat)   |
     | - Lactate (g/L)           |       | - Temperature (degC)      |
     | - Ammonium (mM)           |       | - Agitation rate (rpm)    |
     +---------------------------+       +---------------------------+
```
---

### Dashboard Pages Walkthrough

#### 1. Batch Summary
A comparative table of all cultivation runs with conditional formatting gradients, paired with a Peak VCD ranking bar chart and a VCD vs. Lactate accumulation scatter plot. 
<img width="2308" height="1280" alt="image" src="https://github.com/user-attachments/assets/3cd1deb0-301a-4b32-811d-f495532ed78c" />

#### 2. Run Summary & Cell Growth 
Tracks Viable Cell Density (VCD) and Cell Viability (%) on dual Y-axes with an 80% critical viability reference threshold, paired with dynamic KPI cards for Peak VCD, Time to Peak VCD, and Viability Duration.
<img width="2304" height="1284" alt="image" src="https://github.com/user-attachments/assets/6fff5a18-7eb6-4883-81a8-c3b034567f7d" />

#### 3. Nutrition & Metabolic Profiles (The Chemistry View)
Correlates carbon substrate consumption (Glucose) with byproduct accumulation (Lactate & Ammonium), highlighting the exact hour glucose drops below $1.0\text{ g/L}$ and flagging toxic ammonium spikes above $5.0\text{ mM}$.
<img width="2312" height="1280" alt="image" src="https://github.com/user-attachments/assets/4563bfe8-9790-4fec-89fe-e6ada50144ef" />

#### 4. Inline Sensors & Bioreactor Control 
Monitors continuous bioreactor parameters ($p\text{O}_2$, $p\text{CO}_2$, pH, Temperature, and Agitation rate) along with calculated control ranges.
<img width="2306" height="1286" alt="image" src="https://github.com/user-attachments/assets/b30d4bc3-7e87-4815-97e2-7192a3a9d990" />

---

#### Highlighted DAX Measures

All calculations are centralized in a dedicated `_Measures` table. Below are some examples demonstrating filter context manipulation, tie-breaking aggregation, and string formatting:

```dax
// 1. Time to Peak VCD
Hour_to_Max_VCD = 
VAR MaxVCDVal = [Max_VCD]
RETURN
CALCULATE(
    MIN(Offline_Data[Time (h)]),
    FILTER(
        ALLSELECTED(Offline_Data),
        Offline_Data[VCD Mean (cells/mL)] = MaxVCDVal
    )
)

// 2. Hours to Critical Glucose Limitation (<= 1.0 g/L) 
Hour_to_Glucose_Limit = 
VAR CurrentExperiment = SELECTEDVALUE(Dim_Experiment[Experiment_ID])
RETURN
CALCULATE(
    MIN(Offline_Data[Time (h)]),
    FILTER(
        ALLSELECTED(Offline_Data),
        Offline_Data[Experiment_ID] = CurrentExperiment &&
        Offline_Data[Glucose (g/L)] <= 1.0 &&
        Offline_Data[Glucose (g/L)] <> BLANK()
    )
)

// 3. Dynamic pH Control Range
pH_Control_Range = 
VAR Min_pH = MIN(Inline_Data[pH])
VAR Max_pH = MAX(Inline_Data[pH])
RETURN
FORMAT(Min_pH, "0.00") & " – " & FORMAT(Max_pH, "0.00")
```

---

#### Skills Demonstrated
* **Microsoft Power BI Desktop:** End-to-end report design, custom report-page tooltips, synchronized cross-page slicers, and conditional formatting heatmaps.
* **Power Query (M):** Multi-file folder ingestion, delimiter parsing, and schema pruning.
* **Data Modeling & DAX:** Conformed Star Schema design across mixed time granularities, context transition handling (`CALCULATE`, `FILTER`, `ALLSELECTED`), and variable-driven DAX formulas.
* **Bioprocess Domain Knowledge:** Upstream CHO cell physiology, substrate limitation kinetics, byproduct inhibition, and bioreactor PID control monitoring.

---

#### How to Use This Repository
1. Clone or download this repository.
2. Open the `Bioprocess Analytics Dashboard.pbix` file in **Microsoft Power BI Desktop**.
