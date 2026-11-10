# Bioprocess Analytics Dashboard with PowerBI

### Project Overview
This repository showcases an end-to-end **Bioprocess Monitoring** built in **Microsoft Power BI**. 
Developed as my **first-ever hands-on project in Power BI**, this project translates complex upstream Chinese Hamster Ovary (CHO) cell cultivation data into a dashboard visualization for process insights. 
---
### Dataset
**Source:** Dataset Based on Chinese Hamster Ovary (CHO) Cultivations including Turbidity, Permittivity, O2 and CO2 Measurements
 [(Zenodo: 10.5281/zenodo.20829178)](https://doi.org/10.5281/zenodo.20829178)
* **Scope:** 24 bioreactor cultivation runs tracking both daily offline laboratory samples and continuous high-frequency inline sensor telemetry.
Citations: Uhlendorff, S., Fulek, R., Eimler, J., Pein-Hackelbusch, M., & Frahm, B. (2026). Dataset Based on Chinese Hamster Ovary (CHO) Cultivations including Turbidity, Permittivity, O2 and CO2 Measurements (Version Version v1) [Dataset]. Zenodo. https://doi.org/10.5281/zenodo.20829178
---

## 🎯 Key Bioprocess & Engineering Insights Solved

1. **Cellular Growth & Harvest Timing ("The Biology"):**
   * Pinpointed the exact transition from exponential growth to stationary/decline phases by dynamically calculating **Peak Viable Cell Density ($VCD_{max}$)** and the **Cultivation Hour of Peak VCD**.
   * Tracked **Longevity of High Viability ($\ge 80\%$)** to guide optimal downstream harvest windows before cell lysis releases host-cell proteases.
2. **Root Cause Metabolic Analysis ("The Chemistry"):**
   * Engineered a **Critical Glucose Limitation ($\le 1.0\text{ g/L}$)** tracker rather than waiting for absolute zero depletion—mathematically proving that carbon starvation at $136.0\text{ h}$ in Batch Run 1 directly triggered growth arrest and apoptosis.
   * Monitored **Lactate Metabolic Shifts** (overflow production vs. secondary consumption) alongside **Ammonium ($\text{NH}_4^+$) Toxicity** against the $5.0\text{ mM}$ inhibitory threshold.
3. **Bioreactor Hardware & PID Loop Stability ("The Machine"):**
   * Evaluated continuous inline telemetry (**pH**, **Dissolved Oxygen / $p\text{O}_2$**, **Temperature**, and **Agitation Rate**) to verify automated setpoint control across multi-day cultivations.
4. **High-Throughput Fleet Screening ("The Leaderboard"):**
   * Built an executive comparative matrix with conditional heatmaps to screen all cultivations side-by-side, immediately isolating top-performing "Golden Batches" (e.g., Fed-Batch vs. Batch strategies).

---

## 🏗️ Data Architecture: Conformed Multi-Fact Star Schema

A core data engineering challenge in biopharma Process Analytical Technology (PAT) is aligning **low-frequency offline sampling** (1–2 samples/day) with **high-frequency inline sensor logs** (continuous minute/second readings) without introducing null bloat or breaking time-series visuals.

To solve this in Power BI, I architected a **Conformed Multi-Fact Star Schema**:

```text
                      +-----------------------------+
                      |       Dim_Experiment        |
                      |-----------------------------|
                      | * Experiment_ID (PK)        |
                      +--------------+--------------+
                                     |
                   +-----------------+-----------------+
                   | 1:Many (Single)                   | 1:Many (Single)
                   v                                   v
     +---------------------------+       +---------------------------+
     |     Fact_Offline_Data     |       |     Fact_Inline_Data      |
     |---------------------------|       |---------------------------|
     | * Experiment_ID (FK)      |       | * Experiment_ID (FK)      |
     | - Cultivation_Time_h      |       | - Cultivation_Time_h      |
     | - Viable_Cell_Density     |       | - pH                      |
     | - Viability_Pct           |       | - Dissolved_O2_Pct        |
     | - Glucose_gL              |       | - Dissolved_CO2_Pct       |
     | - Lactate_gL              |       | - Temperature_degC        |
     | - Ammonium_mM             |       | - Agitation_rpm           |
     +---------------------------+       +---------------------------+
```

### ⚡ VertiPaq Memory & File Size Optimization
To compress the high-frequency inline telemetry and keep the `.pbix` file lightweight and responsive:
* **Automated Folder Ingestion:** Combined 12+ raw data files dynamically via Power Query's Folder connector.
* **Signal Band-Pass Filtering:** Filtered out impossible physical sensor dropouts/spikes (e.g., momentary probe calibration spikes in agitation and temperature) in Power Query to protect visual scales and mean KPI accuracy.
* **Columnar Compression:** Pruned unused sensor channels, rounded high-precision floating-point telemetry, and disabled hidden Auto Date/Time tables to maximize Power BI's VertiPaq columnar compression.

---

## 📊 Dashboard Pages Walkthrough

### 1. Run Summary & Cell Growth (The Biology View)
Tracks Viable Cell Density (VCD) and Cell Viability (%) on dual Y-axes with an 80% critical viability reference threshold, paired with dynamic KPI cards for Peak VCD, Time to Peak VCD, and Viability Duration.
> 📸 *(Insert screenshot: `docs/page1_cell_growth.png`)*

### 2. Nutrition & Metabolic Profiles (The Chemistry View)
Correlates carbon substrate consumption (Glucose) with byproduct accumulation (Lactate & Ammonium), highlighting the exact hour glucose drops below $1.0\text{ g/L}$ and flagging toxic ammonium spikes above $5.0\text{ mM}$.
> 📸 *(Insert screenshot: `docs/page2_nutrition_profile.png`)*

### 3. Inline Sensors & Bioreactor Control (The Engineering View)
Monitors continuous bioreactor parameters ($p\text{O}_2$, $p\text{CO}_2$, pH, Temperature, and Agitation RPM) along with calculated control ranges to verify PID loop stability.
> 📸 *(Insert screenshot: `docs/page3_inline_sensors.png`)*

### 4. Multi-Run Screening & Leaderboard (The Executive View)
A fleet-wide comparative matrix with conditional formatting gradients, paired with a Peak VCD ranking bar chart and a Biomass vs. Lactate efficiency scatter plot. Includes **Cross-Page Drillthrough** so users can right-click any batch to inspect its individual run curves.
> 📸 *(Insert screenshot: `docs/page4_leaderboard.png`)*

---

## 🧮 Highlighted DAX Measures

All calculations are centralized in a dedicated `_Measures` table. Below are key examples demonstrating filter context manipulation, tie-breaking aggregation, and string formatting:

```dax
// 1. Time to Peak VCD (Uses MIN to cleanly resolve multi-timepoint plateau ties)
Hours to Peak VCD = 
VAR CurrentExperiment = SELECTEDVALUE(Dim_Experiment[Experiment_ID])
VAR MaxVCDVal = [Peak VCD]
RETURN
CALCULATE(
    MIN(Fact_Offline_Data[Cultivation_Time_h]),
    FILTER(
        ALLSELECTED(Fact_Offline_Data),
        Fact_Offline_Data[Experiment_ID] = CurrentExperiment &&
        Fact_Offline_Data[Viable_Cell_Density] = MaxVCDVal
    )
)

// 2. Hours to Critical Glucose Limitation (<= 1.0 g/L) with Row-Context Preservation
Hours to Glucose <= 1.0 g/L = 
VAR CurrentExperiment = SELECTEDVALUE(Dim_Experiment[Experiment_ID])
RETURN
CALCULATE(
    MIN(Fact_Offline_Data[Cultivation_Time_h]),
    FILTER(
        ALLSELECTED(Fact_Offline_Data),
        Fact_Offline_Data[Experiment_ID] = CurrentExperiment &&
        Fact_Offline_Data[Glucose_gL] <= 1.0 &&
        Fact_Offline_Data[Glucose_gL] <> BLANK()
    )
)

// 3. Dynamic pH Control Range String
pH Control Range = 
VAR Min_pH = MIN(Fact_Inline_Data[pH])
VAR Max_pH = MAX(Fact_Inline_Data[pH])
RETURN
FORMAT(Min_pH, "0.00") & " – " & FORMAT(Max_pH, "0.00")
```

---

## 🛠️ Skills Demonstrated
* **Microsoft Power BI Desktop:** End-to-end report design, custom report-page tooltips, synchronized cross-page slicers, conditional formatting heatmaps, and drillthrough navigation.
* **Power Query (M):** Multi-file folder ingestion, delimiter parsing, sensor noise filtering, and schema pruning.
* **Data Modeling & DAX:** Conformed Star Schema design across mixed time granularities, context transition handling (`CALCULATE`, `FILTER`, `ALLSELECTED`), and variable-driven DAX formulas.
* **Bioprocess Domain Knowledge:** Upstream CHO cell physiology, substrate limitation kinetics, byproduct inhibition, and bioreactor PID control monitoring.

---

## 🚀 How to Use This Repository
1. Clone or download this repository.
2. Open the `.pbix` (or `.pbit` template) file in **Microsoft Power BI Desktop**.
3. Use the **Fleet Leaderboard** page to compare all 24 runs, or right-click any `Experiment_ID` and select **Drill through** to inspect its biological, metabolic, and hardware telemetry in detail.
