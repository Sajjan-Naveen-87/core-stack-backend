# CoRE Stack Mathematical Formulae Reference

This document compiles the mathematical equations, environmental indices, and GIS decision rules implemented in the CoRE Stack backend. It maps each formula to its specific NRM/planning use case and the exact location file within the project structure.

---

## 1. Hydrological Water Budgeting Pipeline

These equations calculate the water balance components (Precipitation, Runoff, Evapotranspiration, Recharge) and the subsequent groundwater fluctuations in individual Microwatersheds (MWS).

### A. Precipitation ($P$)
*   **Formula**: Cumulative sum of JAXA GSMaP hourly satellite rain rates ($mm/hr$) over the target period (14-day fortnight or 365-day hydrological year):
    $$P = \sum_{t=1}^{T} \text{hourlyPrecipRate}_t$$
*   **Use Case**: Primary input representing total water influx from rainfall.
*   **Location File**: [computing/mws/precipitation.py](file:///home/snaveen/Desktop/core-stack-backend/computing/mws/precipitation.py#L98-L121)

### B. Evapotranspiration ($ET$)
*   **Formula**: Daily cumulative evapotranspiration depth ($mm$) computed from NASA FLDAS/GLDAS average evapotranspiration rate (`Evap_tavg`, in $kg/m^2/s$):
    $$ET = \sum_{d=1}^{D} \left( \text{Evap\_tavg}_d \times 86400 \times \text{weight}_d \times 0.1 \right)$$
    *   *Conversion*: $1 \text{ kg/m}^2 \text{ of water} = 1\text{ mm of depth}$. Multiplying the daily average rate by $86400$ seconds converts it to daily depth.
    *   *Weight*: The proportion of days in a month that fall in the current planning period (since FLDAS publishes monthly averages).
    *   *MODIS Scale Factor*: $0.1$.
*   **Use Case**: Calculating water returned to the atmosphere via transpiration and soil evaporation.
*   **Location File**: [computing/mws/evapotranspiration.py](file:///home/snaveen/Desktop/core-stack-backend/computing/mws/evapotranspiration.py#L262-L282)

### C. Slope Percentage ($sp$)
*   **Formula**: Trigonometric tangent of the slope angle ($\theta$, in degrees) derived from SRTM DEM elevation model:
    $$sp = \tan\left(\frac{\theta \times \pi}{180}\right) \times 100$$
*   **Use Case**: Normalizing terrain steepness for Curve Number adjustments and CLART recommendations.
*   **Location Files**:
    *   SCS-CN: [computing/mws/run_off.py](file:///home/snaveen/Desktop/core-stack-backend/computing/mws/run_off.py#L191-L192)
    *   CLART: [computing/clart/clart.py](file:///home/snaveen/Desktop/core-stack-backend/computing/clart/clart.py#L86-L89)

### D. Slope-Adjusted Base Curve Number ($CN_{2a}$)
*   **Formula**: SCS Curve Number adjusted for steep slopes using a custom Google Earth Engine (GEE) expression:
    $$CN_3 = CN_2 \cdot e^{0.00673 \cdot (100 - CN_2)}$$
    $$CN_{2a} = \left(\frac{CN_3 - CN_2}{3}\right) \cdot (1 - 2 \cdot e^{-13.86 \cdot sp}) + CN_2$$
    *   Where $CN_2$ is the base Curve Number derived from the intersection of Dynamic World LULC and Hydrologic Soil Groups.
*   **Use Case**: Correcting runoff estimates to prevent underestimating runoff on steep slopes.
*   **Location File**: [computing/mws/run_off.py](file:///home/snaveen/Desktop/core-stack-backend/computing/mws/run_off.py#L286-L320)

### E. Dry ($CN_{1a}$) and Wet ($CN_{3a}$) AMC Curve Numbers
*   **Formula**: Adjusting curve numbers for dry/wet Antecedent Moisture Conditions (AMC):
    $$CN_{1a} = \frac{4.2 \cdot CN_{2a}}{10 - 0.058 \cdot CN_{2a}}$$
    $$CN_{3a} = \frac{23 \cdot CN_{2a}}{10 + 0.13 \cdot CN_{2a}}$$
*   **Use Case**: Calculating potential maximum retention thresholds based on dynamic soil moisture conditions.
*   **Location File**: [computing/mws/run_off.py](file:///home/snaveen/Desktop/core-stack-backend/computing/mws/run_off.py#L322-L332)

### F. Potential Maximum Retention ($S_1, S_2, S_3$)
*   **Formula**: Maximum retention depth ($mm$) for AMC I, II, and III conditions:
    $$sr_1 = \frac{25400}{CN_{1a}} - 254$$
    $$sr_2 = \frac{25400}{CN_{2a}} - 254$$
    $$sr_3 = \frac{25400}{CN_{3a}} - 254$$
*   **Use Case**: Baseline storage capacity indicator for daily soil infiltration potential.
*   **Location File**: [computing/mws/run_off.py](file:///home/snaveen/Desktop/core-stack-backend/computing/mws/run_off.py#L334-L350)

### G. Soil Moisture Deficit / Antecedent Storage ($M_1, M_2, M_3$)
*   **Formula**: Current daily soil moisture tracking from the 5-day cumulative antecedent rainfall ($P_5$, in $mm$):
    $$M = 0.5 \cdot \left(-S + \sqrt{S^2 + 4 \cdot P_5 \cdot S}\right)$$
*   **Use Case**: Determining real-time soil saturation before computing daily runoff.
*   **Location File**: [computing/mws/run_off.py](file:///home/snaveen/Desktop/core-stack-backend/computing/mws/run_off.py#L371-L400)

### H. Daily Surface Runoff ($Q$)
*   **Formula**: Pixel-level daily runoff depth ($mm$) using the slope-adjusted SCS-CN algorithm:
    *   **Dry AMC (AMC I)** (If $P_5 \le 35\text{ mm}$ and $P \ge 0.2 \cdot sr_1$):
        $$Q = \frac{(P - 0.2 \cdot sr_1) \cdot (P - 0.2 \cdot sr_1 + M_1)}{P + 0.8 \cdot sr_1 + M_1}$$
    *   **Normal AMC (AMC II)** (If $35\text{ mm} < P_5 \le 52.5\text{ mm}$ and $P \ge 0.2 \cdot sr_2$):
        $$Q = \frac{(P - 0.2 \cdot sr_2) \cdot (P - 0.2 \cdot sr_2 + M_2)}{P + 0.8 \cdot sr_2 + M_2}$$
    *   **Wet AMC (AMC III)** (If $P_5 > 52.5\text{ mm}$ and $P \ge 0.2 \cdot sr_3$):
        $$Q = \frac{(P - 0.2 \cdot sr_3) \cdot (P - 0.2 \cdot sr_3 + M_3)}{P + 0.8 \cdot sr_3 + M_3}$$
    *   *Otherwise*: $Q = 0$.
    *   The cumulative sum of pixel-level daily runoffs is divided by the total microwatershed area.
*   **Use Case**: Runoff estimation to design watershed storage capacities.
*   **Location File**: [computing/mws/run_off.py](file:///home/snaveen/Desktop/core-stack-backend/computing/mws/run_off.py#L410-L459)

### I. Net Groundwater Recharge ($\Delta G$)
*   **Formula**: Basic water balance equation per microwatershed geometry:
    $$\Delta G = P - Q - ET$$
*   **Use Case**: Calculating net groundwater replenishment in $mm$.
*   **Location File**: [computing/mws/delta_g.py](file:///home/snaveen/Desktop/core-stack-backend/computing/mws/delta_g.py#L140-L150)

### J. Weighted Average Aquifer Specific Yield ($S_y$)
*   **Formula**: Spatially-weighted yield fraction based on intersecting CGWB principal aquifer polygons:
    $$S_y = \sum_{j=1}^{J} \left( \frac{\text{Area}_{\text{intersection}, j}}{\text{Area}_{\text{MWS}}} \times \text{Yield\_fraction}_j \right)$$
    *   *Specific Yield Mappings*: CGWB descriptive yield strings map to fraction rates:
        *   `Upto 1%` $\rightarrow 0.01$
        *   `Upto 1.5%` / `1-1.5%` $\rightarrow 0.015$
        *   `Upto 2%` / `1-2%` / `1.5-2%` $\rightarrow 0.02$
        *   `Upto 2.5%` / `1-2.5` $\rightarrow 0.025$
        *   `Upto 3%` / `2-3%` $\rightarrow 0.03$
        *   `Upto 3.5%` $\rightarrow 0.035$
        *   `Upto 4%` $\rightarrow 0.04$
        *   `Upto 5%` $\rightarrow 0.05$
        *   `6 - 8%` / `Upto 8%` $\rightarrow 0.08$
        *   `6 - 10%` / `8 - 10%` $\rightarrow 0.10$
        *   `6 - 12%` / `8 - 12%` $\rightarrow 0.12$
        *   `6 - 15%` / `8 - 15%` / `Upto 15%` $\rightarrow 0.15$
        *   `6 - 16%` / `8 - 16%` $\rightarrow 0.16$
        *   `8 - 18%` $\rightarrow 0.18$
        *   `8 - 20%` $\rightarrow 0.20$
*   **Use Case**: Modeling geological storage characteristics of the subterranean aquifer.
*   **Location File**: [computing/mws/well_depth.py](file:///home/snaveen/Desktop/core-stack-backend/computing/mws/well_depth.py#L121-L139)

### K. Predicted Well Depth Fluctuation ($wd$)
*   **Formula**: Vertical water table movement ($m$) calculated via yield division:
    $$wd = \frac{\Delta G}{S_y \times 1000}$$
*   **Use Case**: Translating recharge volume into vertical well water level changes in meters.
*   **Location File**: [computing/mws/well_depth.py](file:///home/snaveen/Desktop/core-stack-backend/computing/mws/well_depth.py#L158)

---

## 2. Composite Drought Assessment

These equations standardise meteorological and crop stress parameters over historical baselines to define drought triggers.

### A. Vegetation Condition Index ($VCI$)
*   **Formula**: Standardisation of current MODIS NDVI and NDWI over long-term extremes (since 2000) inside crop zones:
    $$VCI_{\text{pixel}} = \min\left( \frac{NDVI - NDVI_{min}}{NDVI_{max} - NDVI_{min}}, \frac{NDWI - NDWI_{min}}{NDWI_{max} - NDWI_{min}} \right) \times 100$$
    *   *Microwatershed average VCI* is calculated over cropping pixel counts:
        $$VCI_{\text{MWS}} = \frac{\sum_{ROI} (VCI_{\text{pixel}} \times \text{cropping\_mask})}{\sum_{ROI} \text{cropping\_mask}}$$
*   **Use Case**: Monitoring crop health status relative to historical seasons.
*   **Location File**: [computing/drought/generate_layers.py](file:///home/snaveen/Desktop/core-stack-backend/computing/drought/generate_layers.py#L852-L890)

### B. Moisture Adequacy Index ($MAI$)
*   **Formula**: Ratio of cumulative Evapotranspiration ($ET$) to Potential Evapotranspiration ($PET$) inside crop-sown boundaries over a monthly 28-day window:
    $$MAI = \frac{\sum_{ROI} (ET_{\text{cum}} \times \text{cropping\_mask})}{\sum_{ROI} (PET_{\text{cum}} \times \text{cropping\_mask})} \times 100$$
*   **Use Case**: Tracking crop water supply adequacy.
*   **Location File**: [computing/drought/generate_layers.py](file:///home/snaveen/Desktop/core-stack-backend/computing/drought/generate_layers.py#L998-L1007)

### C. Standardized Precipitation Index ($SPI-1$)
*   **Formula**: Monthly precipitation deviation standardized against long-term historical mean ($\mu$) and standard dev ($\sigma$) since 1981:
    $$SPI = \frac{P_{28\text{-day}} - \mu_{28\text{-day}}}{\sigma_{28\text{-day}}}$$
*   **Use Case**: Characterizing meteorological rainfall anomalies.
*   **Location File**: [computing/drought/generate_layers.py](file:///home/snaveen/Desktop/core-stack-backend/computing/drought/generate_layers.py#L692-L698)

---

## 3. CLART Decision Matrix

This matrix assigns land treatments based on hydrological recharge potential overlays and topography steepness.

### A. Recharge Potential ($rp$)
*   **Formula**: Product of classified GIS layer scores:
    $$rp = dd\_score \times lin\_score \times lith\_score$$
    *   *Lineament score* ($lin\_score$): 10 if present, 1 if absent.
    *   *Drainage density score* ($dd\_score$): Normalized between 0 and 1, then classified:
        *   $dd_{\text{norm}} \le 0.334 \rightarrow 1$ (Low density)
        *   $0.334 < dd_{\text{norm}} \le 0.667 \rightarrow 2$ (Medium density)
        *   $dd_{\text{norm}} > 0.667 \rightarrow 3$ (High density)
    *   *Lithology score* ($lith\_score$): Infiltration capacity class based on aquifer RIF values:
        *   $RIF < 10 \rightarrow 3$ (Low permeability)
        *   $10 \le RIF \le 15 \rightarrow 2$ (Medium permeability)
        *   $RIF > 15 \rightarrow 1$ (High permeability)
    *   *Recharge potential classes*:
        *   $rp \in \{1, 2, 10, 20, 30, 40, 60, 90\} \rightarrow 1$ (High Potential)
        *   $rp \in \{3, 4\} \rightarrow 2$ (Medium Potential)
        *   $rp \in \{6, 9\} \rightarrow 3$ (Low Potential)
        *   *Else* $\rightarrow 0$
*   **Use Case**: Defining physical sub-surface absorption potential maps.
*   **Location Files**:
    *   Main CLART RP logic: [computing/clart/clart.py](file:///home/snaveen/Desktop/core-stack-backend/computing/clart/clart.py#L128-L164)
    *   Lithology RIF match: [computing/clart/lithology.py](file:///home/snaveen/Desktop/core-stack-backend/computing/clart/lithology.py#L125-L133)

### B. CLART Recommendations Rules
*   **Formula**: Recommendation class (1-5) derived from $rp$ and local maximum slope percentage ($max\_sp$):
    *   **Class 1**: Recharge Potential = 1 and $sp \in [0, 0.20 \times max\_sp]$
    *   **Class 2**: Recharge Potential = 2 and $sp \in [0, 0.25 \times max\_sp]$
    *   **Class 3**: Recharge Potential = 3 and $sp \in [0, 0.20 \times max\_sp]$
    *   **Class 4**: Recharge Potential $\in \{1, 2, 3\}$ and $sp \in [0.25 \times max\_sp, 0.30 \times max\_sp]$
    *   **Class 5**: Recharge Potential $\in \{1, 2, 3\}$ and $sp > 0.30 \times max\_sp$
*   **Use Case**: Directing automated check-dam, farm pond, contour trench, and gully plug placements.
*   **Location File**: [computing/clart/clart.py](file:///home/snaveen/Desktop/core-stack-backend/computing/clart/clart.py#L174-L215)
