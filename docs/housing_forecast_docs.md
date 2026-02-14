# CommunityScale Housing Forecast: Technical Documentation

## Table of Contents

- [Recommended Citation](#recommended-citation)
- [Methodologies](#methodologies)
  - [Population Forecast Methodology](#population-forecast-methodology)
  - [Housing Affordability Methodology](#housing-affordability-methodology)
  - [Household Growth by Income Group Methodology](#household-growth-by-income-group-methodology)
  - [10-Year Housing Production Target Methodology](#10-year-housing-production-target-methodology)
- [Data Sources](#data-sources)
- [Bibliography](#bibliography)
- [Glossary of Terms](#glossary-of-terms)

---

## Recommended Citation

**Recommended citation for the CommunityScale Housing Forecast App:**

> CommunityScale Housing Forecast (2025). Housing needs analysis and population projection tool. Available at: https://app.communityscale.io


```bibtex
@misc{communityscale2025,
  title = {CommunityScale Housing Forecast},
  year = {2025},
  howpublished = {\url{https://app.communityscale.io}},
  note = {Housing needs analysis and population projection tool}
}
```

---

## Methodologies

This section provides detailed technical documentation of the analytical methodologies employed in the CommunityScale Housing Forecast application. Each methodology is designed to produce robust, reproducible results grounded in established demographic and economic analysis techniques. The methodologies integrate data from the U.S. Census Bureau, HUD, FRED, and commercial real estate data providers to provide comprehensive housing needs assessments across different geographic scales and income levels.

### Population Forecast Methodology

The CommunityScale population forecasting system employs a **cohort-component method** combined with **ARIMA time series forecasting** to generate demographically detailed population projections by age and sex. This methodology has been widely used in demographic research and official population projections due to its ability to capture age-specific demographic processes.

#### Overview

The cohort-component method divides the population into age groups (cohorts) and tracks how each group changes over time through:
- **Fertility**: Birth rates by age of mother
- **Mortality**: Survival rates by age and sex
- **Migration**: Net migration effects on each age cohort

The system analyzes historical patterns from the American Community Survey (ACS) 2010-2023 to predict future population changes using Leslie matrix population projection combined with ARIMA-forecasted demographic parameters.

#### Data Preparation

Population data is extracted from **ACS Table B01001** (Sex by Age), which provides population counts by single-year and 5-year age groups for males and females. The data is aggregated into 18 standardized age bins representing 5-year age cohorts:

- Age Bin 1: 0-4 years
- Age Bins 2-10: 5-9, 10-14, ..., 45-49 years
- Age Bins 11-17: 50-54, ..., 80-84 years
- Age Bin 18: 85+ years

This binning structure aligns with Census reporting categories and enables tracking of childbearing-age populations (ages 15-49, bins 4-10) for fertility calculations.

#### Fertility Rate Calculation

Annual fertility rates are calculated as:

$$
F_t = \frac{P_{t,0-4}}{W_{t,15-49}}
$$

where:
- $F_t$ = fertility rate in year $t$
- $P_{t,0-4}$ = population aged 0-4 years (newborns cohort)
- $W_{t,15-49}$ = female population aged 15-49 years

Future births are projected using forecasted fertility rates and projected female populations:

$$
\begin{align}
B_{t+5}^{female} &= F_{t+5} \times \sum_{i=4}^{10} W_{t+5,i} \times 0.487 \\
B_{t+5}^{male} &= F_{t+5} \times \sum_{i=4}^{10} W_{t+5,i} \times 0.513
\end{align}
$$

where 0.487 and 0.513 represent the proportion of female and male births respectively, based on the sex ratio at birth.

When fertility rates exhibit high volatility (coefficient of variation > 0.3), the system uses the 25th percentile of historical rates as a constant forecast rather than ARIMA extrapolation to avoid unrealistic projections.

#### Cohort Component Ratios and Differences

The system calculates two alternative measures of cohort change:

**Cohort Change Ratio (CCR)** - multiplicative approach:

$$
CCR_{i,t} = \frac{P_{i+1,t}}{P_{i,t-5}}
$$

**Cohort Change Difference (CCD)** - additive approach:

$$
CCD_{i,t} = P_{i+1,t} - P_{i,t-5}
$$

where:
- $P_{i,t}$ = population in age bin $i$ at time $t$
- The subscript $i+1$ represents aging into the next cohort over 5 years

These metrics capture the combined effects of mortality and migration on each age cohort. For the oldest age group (85+), a special calculation accounts for overlap:

$$
CCR_{17} = \frac{P_{18,t}}{P_{17,t-5} + P_{18,t-5}}
$$

**Selection Logic**: The system selects CCR (multiplicative) for declining populations and CCD (additive) for growing populations, as these methods perform better under their respective demographic conditions.

#### ARIMA Time Series Forecasting

Cohort change parameters and fertility rates are forecasted using **ARIMA(0,1,1)** models. This specification includes:
- **p = 0**: No autoregressive terms
- **d = 1**: First-order differencing to achieve stationarity
- **q = 1**: Moving average of order 1

The ARIMA(0,1,1) model is represented as:

$$
\Delta Y_t = \mu + \epsilon_t + \theta_1 \epsilon_{t-1}
$$

where:
- $\Delta Y_t = Y_t - Y_{t-1}$ (first difference)
- $\mu$ = drift parameter
- $\epsilon_t$ = white noise error term
- $\theta_1$ = moving average parameter

Forecasts extend 15 years beyond the target year (rounded to the nearest 5-year interval) with 80% confidence intervals calculated as:

$$
CI_{80} = \hat{Y}_t \pm 1.28 \times \sigma_{residual}
$$

where the subscript indicates an 80% confidence interval.

#### Leslie Matrix Population Projection

Population projections are generated using Leslie matrices, a standard demographic tool for age-structured population modeling. For each sex, an 18×18 Leslie matrix $\mathbf{L}$ is constructed with:

- **First row**: Fertility components (births from reproductive-age women)
- **Subdiagonal elements** $L_{i,i-1}$: Cohort change ratios or differences representing survival and migration
- **Element** $L_{17,17}$: Special handling for the open-ended 85+ age group

**For CCR (multiplicative) method:**

$$
\mathbf{P}_{t+5} = \mathbf{L}_t \cdot \mathbf{P}_t
$$

**For CCD (additive) method:**

$$
\mathbf{P}_{t+5} = \mathbf{L}_t + \mathbf{P}_t
$$

where:
- $\mathbf{P}_t$ = population vector (18×1) at time $t$
- $\mathbf{L}_t$ = Leslie matrix with forecasted demographic parameters

The projection advances in 5-year steps for 6 iterations (30 years total), updating the Leslie matrix at each step with newly forecasted parameters.

#### Uncertainty Quantification

Three scenarios are generated for each projection:

1. **Mean forecast**: Using point estimates from ARIMA models
2. **Low bound (80% CI)**: Using lower confidence interval estimates
3. **High bound (80% CI)**: Using upper confidence interval estimates

This provides a probabilistic range for population projections that accounts for uncertainty in demographic parameter forecasts.

#### Annual Interpolation

Since demographic processes operate on 5-year intervals but annual estimates are needed, the system applies **PCHIP (Piecewise Cubic Hermite Interpolating Polynomial)** interpolation to generate smooth annual population estimates. PCHIP preserves monotonicity and avoids oscillations that can occur with higher-order polynomial interpolation.

#### Data Sources

- **American Community Survey (ACS) 2010-2023**: Table B01001 (Sex by Age)
- **ACS 5-Year Estimates**: Used for geographic areas with populations < 65,000

---

### Housing Affordability Methodology

The housing affordability analysis calculates the income required to afford the typical home price in each geographic area, incorporating all major components of homeownership costs.

#### Monthly Housing Cost Components

Total monthly housing cost is calculated as:

$$
C_{monthly} = M + T + I_p + I_m
$$

where:
- $M$ = Monthly mortgage payment
- $T$ = Monthly property tax
- $I_p$ = Monthly property insurance
- $I_m$ = Monthly mortgage insurance (if applicable)

#### Mortgage Payment Calculation

The monthly mortgage payment uses the standard amortization formula:

$$
M = L \times \frac{r(1+r)^n}{(1+r)^n - 1}
$$

where:
- $L$ = Loan amount = $P \times (1 - d)$
- $P$ = Home purchase price
- $d$ = Down payment percentage (default 20%)
- $r$ = Monthly interest rate = $R_{annual} / 12$
- $n$ = Number of payments = 360 (30-year mortgage)

#### Property Tax

Monthly property tax is calculated as:

$$
T = \frac{P \times \tau}{12}
$$

where $\tau$ is the effective property tax rate, calculated from ACS data as:

$$
\tau = \frac{\text{Selected Monthly Owner Costs (B25103)}}{\text{Median Home Value (B25077)}}
$$

**Data Sources:**
- **ACS Table B25103**: Selected Monthly Owner Costs
- **ACS Table B25077**: Median Home Value

#### Property Insurance

Monthly property insurance is calculated as:

$$
I_p = \frac{P \times \rho}{12}
$$

where $\rho$ is the annual insurance rate.

**Data Sources:**
- Geographic-level insurance rates aggregated from ZIP code-level data
- National average insurance rate used as fallback when local data unavailable

#### Mortgage Insurance (PMI)

When down payment < 20%, private mortgage insurance is required:

$$
I_m = \begin{cases}
\frac{L \times 0.005}{12} & \text{if } d < 0.20 \\
0 & \text{otherwise}
\end{cases}
$$

#### Required Income Calculation

The required annual income applies the standard 30% housing cost burden threshold:

$$
I_{required} = \frac{C_{monthly} \times 12}{0.30}
$$

This assumes that housing costs should not exceed 30% of gross household income, a widely-used affordability standard.

#### Time Series Construction

The affordability analysis generates monthly time series from 2010 to present by integrating:

1. **Home Values**: Commercial home value index - monthly data for middle-tier single-family homes and condos (33rd-67th percentile price range)

2. **Mortgage Rates**: 30-year fixed mortgage rates from FRED (Federal Reserve Economic Data) series MORTGAGE30US

3. **Median Income**: ACS Table B19113 (Median Family Income), available annually. Monthly values are interpolated using:
   - Historical ACS data (2010-2023)
   - Per capita wage growth rates from FRED for extrapolation
   - Compound monthly growth rate: $(1 + r_{annual})^{1/12} - 1$

4. **Property Tax Rates**: Calculated annually from ACS data, held constant within each year

The final time series contains:
- `date`: Month-end date
- `median_income`: Median family income (annual, inflation-adjusted)
- `required_income`: Annual income required to afford typical home price at 30% housing cost burden

#### Contributing Factors to Affordability

The methodology captures the following factors affecting housing affordability:

1. **Home Prices**: Commercial home value indices reflect market conditions and housing supply/demand
2. **Interest Rates**: FRED 30-year mortgage rates capture financing costs
3. **Property Taxes**: Local tax rates from ACS reflect fiscal policy impacts
4. **Insurance Costs**: Geographic variation in property insurance premiums
5. **Income Growth**: Median family income trends from Census data
6. **Down Payment Requirements**: 20% standard, with PMI penalty for lower down payments
7. **Loan Terms**: 30-year fixed-rate mortgage assumptions

#### Data Sources

| Component | Source | Table/Series | Frequency |
|-----------|--------|--------------|-----------|
| Home Values | Commercial Data Provider | Proprietary home value index (middle tier, smoothed, seasonally adjusted) | Monthly |
| Mortgage Rates | FRED | MORTGAGE30US | Weekly |
| Median Income | U.S. Census ACS | B19113 | Annual |
| Income Growth | FRED | Per capita wage growth | Annual |
| Property Tax | U.S. Census ACS | B25103, B25077 | Annual |
| Insurance Rates | ZIP-level aggregates | Proprietary insurance data | Static |

---

### Household Growth by Income Group Methodology

This methodology forecasts household growth segmented by Area Median Income (AMI) categories, enabling analysis of housing needs across different income levels.

#### AMI Group Definitions

Households are classified into six AMI groups based on their income relative to the HUD-defined Area Median Income:

| AMI Group | Income Range | Label |
|-----------|-------------|-------|
| 1 | 0-30% AMI | Very Low Income |
| 2 | 30-60% AMI | Low Income |
| 3 | 60-80% AMI | Moderate Income |
| 4 | 80-100% AMI | Middle Income |
| 5 | 100-120% AMI | Above Median |
| 6 | >120% AMI | High Income |

These thresholds align with HUD income limit categories used in affordable housing policy.

#### Census Income Data Rebinning

ACS Table B19001 provides household income in 16 predefined bins:

$$
\text{Bins: } [0, 10k), [10k, 15k), [15k, 20k), ..., [200k, \infty)
$$

These must be rebinned to the 6 AMI-based categories, which have dollar thresholds that vary by geography and year:

$$
\text{AMI Threshold}_{i,g,t} = \text{AMI}_{g,t} \times \alpha_i
$$

where:
- $\text{AMI}_{g,t}$ = Area Median Income for geography $g$ in year $t$ (from HUD)
- $\alpha_i$ = AMI percentage threshold (0.30, 0.60, 0.80, 1.00, 1.20)

**Rebinning Algorithm**: Households are redistributed from Census bins to AMI bins using proportional allocation based on overlap between bin ranges:

$$
H_{new,j} = \sum_{i} H_{census,i} \times \frac{\text{overlap}(B_{census,i}, B_{AMI,j})}{\text{width}(B_{census,i})}
$$

where:
- $H_{new,j}$ = households in AMI bin $j$
- $H_{census,i}$ = households in Census bin $i$
- $\text{overlap}()$ = dollar range overlap between bins
- $\text{width}()$ = bin width

#### Inflation Adjustment

Historical AMI thresholds are adjusted for inflation to maintain consistent real income categories:

$$
\text{Threshold}_{real,t} = \frac{\text{Threshold}_{nominal,t}}{\text{CPI}_t / \text{CPI}_{base}}
$$

This ensures that an "80% AMI" household in 2010 represents the same purchasing power as in 2023.

#### Population-to-Household Conversion

Total household counts are derived from population forecasts using PUMS (Public Use Microdata Sample) data:

$$
H_{type,t} = \frac{\sum_{age,sex} P_{age,sex,t} \times \text{Prop}_{age,sex,type}}{\text{Ratio}_{pop/hh,type}}
$$

where:
- $P_{age,sex,t}$ = population forecast by age and sex
- $\text{Prop}_{age,sex,type}$ = proportion of (age, sex) in household type
- $\text{Ratio}_{pop/hh,type}$ = population-to-household ratio for household type

**Household Types** (13 categories based on size and children):
- 1-person households (no children)
- 2-person households (with/without children)
- 3-7+ person households (classified by number of adults and children)

**PUMS Data Sources:**
- **2019-2023 ACS 5-Year PUMS**: Household and person microdata files
- Weighted by WGTP (household weight) and PWGTP (person weight)
- Aggregated to PUMA (Public Use Microdata Area) geographies

#### Income Distribution Forecasting

The share of households in each AMI group is forecasted using:

1. **Historical Trend Estimation**: Linear regression of income shares over time

$$
S_{i,t} = \beta_0 + \beta_1 \times t + \epsilon_t
$$

2. **Constraint and Normalization**: Forecasted shares are clipped to [0, 1] and renormalized:

$$
S_{i,t}^{normalized} = \frac{\max(0, \min(1, S_{i,t}))}{\sum_{j=1}^{6} \max(0, \min(1, S_{j,t}))}
$$

3. **Blended Forecast**: Combine base year distribution with trend forecast:

$$
S_{i,t}^{blend} = (1 - w) \times S_{i,base} + w \times S_{i,t}^{trend}
$$

where $w = 0.30$ (30% weight on trend, 70% on base year) to dampen extreme extrapolations.

#### Final Household Projection by AMI Group

$$
H_{AMI,i,t} = S_{i,t}^{blend} \times H_{total,t}
$$

where $H_{total,t}$ is the total household forecast from population projections.

#### Data Sources

- **ACS Table B19001**: Household Income in the Past 12 Months (16 bins)
- **HUD Income Limits**: Area Median Income by geography and year (2010-2025)
- **ACS PUMS 2019-2023 5-Year**: Household and person microdata
- **FRED**: Per capita wage growth for income trend analysis
- **Population Forecasts**: From Leslie Matrix methodology (see Population Forecast section)

---

### 10-Year Housing Production Target Methodology

The housing production target represents the total number of new housing units needed over a 10-year period to accommodate projected growth while addressing existing market imbalances. This comprehensive approach accounts for six distinct components of housing need.

#### Component 1: Net Household Growth

**Formula:**

$$
N_{growth} = H_{t+10} - H_t
$$

where:
- $H_t$ = current household count
- $H_{t+10}$ = projected household count in 10 years

Household growth is calculated from population forecasts (described above) using PUMS-derived population-to-household ratios by household type. This baseline demand represents new households formed through population growth, changes in household formation rates, and demographic shifts.

**Data Sources:**
- Population forecasts (ACS B01001, Leslie Matrix projections)
- PUMS household composition ratios

#### Component 2: Replacement Housing

Replacement housing accounts for units lost to demolition, disaster, or conversion to non-residential use. The annual replacement rate is calculated from historical patterns in ACS year-built data.

**Formula:**

$$
N_{replacement} = r_{replacement} \times H_{existing} \times 10
$$

where:
- $r_{replacement}$ = annual replacement rate
- $H_{existing}$ = current housing stock

The replacement rate is estimated from **ACS Table B25034** (Year Structure Built) by identifying the rate at which older housing stock exits the market based on cohort survival analysis.

**Data Sources:**
- **ACS Table B25034**: Year Structure Built (11 time period categories)

#### Component 3: Vacancy Adjustments

Vacancy adjustments ensure healthy market functioning by comparing current vacancy rates to minimum stability targets.

**Target Vacancy Rates:**
- **Owner-occupied**: 1.5%
- **Renter-occupied**: 7.4%

These targets are derived from historical market equilibrium analysis. Owner-occupied units require lower vacancy to enable normal market turnover for home sales. Rental units require higher vacancy to allow reasonable choice and mobility for renters.

**Formula:**

$$
N_{vacancy} = \begin{cases}
H_{owner} \times (\tau_{owner,target} - \tau_{owner,actual}) & \text{if } \tau_{owner,actual} < \tau_{owner,target} \\
0 & \text{otherwise}
\end{cases}
$$

$$
+ \begin{cases}
H_{renter} \times (\tau_{renter,target} - \tau_{renter,actual}) & \text{if } \tau_{renter,actual} < \tau_{renter,target} \\
0 & \text{otherwise}
\end{cases}
$$

where:
- $\tau_{target}$ = target vacancy rate (1.5% owner, 7.4% renter)
- $\tau_{actual}$ = current vacancy rate
- $H_{owner}$, $H_{renter}$ = existing owner and renter housing stock

When current rates fall below these thresholds, additional units are needed to restore market fluidity.

**Source:** Belsky, E. S., Drew, R. B., & McCue, D. (2007)

**Data Sources:**
- **ACS Table B25002**: Occupancy Status
- **ACS Table B25003**: Tenure (Owner/Renter)

#### Component 4: Overcrowding Adjustments

Overcrowding is defined as more than 1.0 person per room. When local overcrowding rates exceed national averages, additional housing units are prescribed to provide adequate housing options.

**Formula:**

$$
N_{overcrowding} = \begin{cases}
H_{overcrowded} - (H_{total} \times r_{national}) & \text{if } r_{local} > r_{national} \\
0 & \text{otherwise}
\end{cases}
$$

where:
- $r_{local}$ = local overcrowding rate
- $r_{national}$ = national overcrowding rate (benchmark)
- $H_{overcrowded}$ = households with >1.0 person per room
- $H_{total}$ = total households

**Data Sources:**
- **ACS Tables**: Household size and rooms in housing unit
- National overcrowding benchmarks from ACS national estimates

#### Component 5: Substandard Housing Adjustments

Substandard housing is defined as units lacking complete plumbing or kitchen facilities. Similar to overcrowding, when local rates exceed national norms, additional units are needed.

**Formula:**

$$
N_{substandard} = \begin{cases}
H_{substandard} - (H_{total} \times s_{national}) & \text{if } s_{local} > s_{national} \\
0 & \text{otherwise}
\end{cases}
$$

where:
- $s_{local}$ = local substandard housing rate
- $s_{national}$ = national substandard rate (benchmark)
- $H_{substandard}$ = units lacking complete facilities

**Data Sources:**
- **ACS Tables**: Complete plumbing facilities, complete kitchen facilities
- National benchmarks from ACS national estimates

#### Component 6: Total Production Target

The total production target sums all components:

$$
N_{total} = N_{growth} + N_{replacement} + N_{vacancy} + N_{overcrowding} + N_{substandard}
$$

This comprehensive metric addresses:
- **Growth**: New household formation
- **Replacement**: Maintaining existing stock
- **Market Health**: Adequate vacancy for market function
- **Quality**: Reducing overcrowding and substandard conditions

---

## Data Sources

### Primary Data Sources

#### U.S. Census Bureau

**American Community Survey (ACS) 5-Year Estimates**
- **Coverage**: 2010-2023 (14 survey years)
- **Geography**: All geographies with population ≥ 65,000; PUMA-level for microdata
- **Update Frequency**: Annual release

**Key Tables:**
- **B01001**: Sex by Age (population by age cohorts and sex)
- **B19001**: Household Income in the Past 12 Months (16 income bins)
- **B19113**: Median Family Income
- **B25001**: Housing Units (total count)
- **B25002**: Occupancy Status
- **B25003**: Tenure (Owner/Renter Occupied)
- **B25010**: Average Household Size
- **B25034**: Year Structure Built
- **B25077**: Median Home Value
- **B25103**: Mortgage Status and Selected Monthly Owner Costs

**ACS Public Use Microdata Sample (PUMS)**
- **Coverage**: 2019-2023 5-Year PUMS
- **Files**: Household and person microdata
- **Weights**: WGTP (household), PWGTP (person)
- **Purpose**: Household composition ratios, population-to-household conversions

**TIGER/Line Shapefiles**
- **Vintage**: 2023
- **Layers**: State, County, County Subdivision, Place, Census Tract, PUMA, CBSA, ZIP Code Tabulation Area
- **Format**: PostGIS geometry in PostgreSQL database

#### HUD (U.S. Department of Housing and Urban Development)

**Income Limits Data**
- **Years**: 2010-2025
- **Content**: Area Median Income (AMI) and income limit thresholds
- **Household Sizes**: 1-8 persons
- **Thresholds**: Extremely Low Income (30% AMI), Very Low Income (50% AMI), Low Income (80% AMI), Median Family Income (100% AMI)
- **Source**: https://www.huduser.gov/portal/datasets/il.html

#### Federal Reserve Economic Data (FRED)

**Mortgage Rates**
- **Series**: MORTGAGE30US
- **Description**: 30-Year Fixed Rate Mortgage Average in the United States
- **Frequency**: Weekly
- **Source**: Freddie Mac Primary Mortgage Market Survey
- **API**: https://fred.stlouisfed.org/

**Per Capita Income Growth**
- **Purpose**: Income extrapolation and inflation adjustment
- **Frequency**: Annual

#### Commercial Real Estate Data

**Home Value Index**
- **Series**: Middle-tier Single-Family and Condo (33rd-67th percentile)
- **Smoothing**: Seasonally adjusted, smoothed
- **Frequency**: Monthly (end of month)
- **Geography**: County, County Subdivision, Place (where available)
- **Source**: Proprietary commercial real estate data provider

#### Insurance Rate Data

**Property Insurance Rates**
- **Aggregation**: ZIP code level aggregated to Census geographies
- **Content**: Average annual property insurance rates
- **Purpose**: Housing cost calculations

---

## Bibliography

Belsky, E. S., Drew, R. B., & McCue, D. (2007). *Projecting the Underlying Demand for New Housing Units: Inferences from the Past, Assumptions about the Future* (W07-7). Joint Center for Housing Studies, Harvard University. Retrieved from https://www.jchs.harvard.edu/research-areas/working-papers/projecting-underlying-demand-new-housing-units-inferences-past

Box, G. E. P., Jenkins, G. M., Reinsel, G. C., & Ljung, G. M. (2015). *Time Series Analysis: Forecasting and Control* (5th ed.). Wiley.

Leslie, P. H. (1945). On the use of matrices in certain population mathematics. *Biometrika*, 33(3), 183-212. https://doi.org/10.1093/biomet/33.3.183

Rayer, S., & Smith, S. K. (2010). Factors affecting the accuracy of subcounty population forecasts. *Journal of Planning Education and Research*, 30(2), 147-161. https://doi.org/10.1177/0739456X10380056

U.S. Census Bureau. (ongoing). *American Community Survey 5-Year Estimates*. Retrieved from https://www.census.gov/programs-surveys/acs

U.S. Census Bureau. (ongoing). *Public Use Microdata Sample (PUMS)*. Retrieved from https://www.census.gov/programs-surveys/acs/microdata.html

U.S. Department of Housing and Urban Development. (ongoing). *Income Limits Documentation System*. Retrieved from https://www.huduser.gov/portal/datasets/il.html

Wilson, T., Grossman, I., Alexander, M., Rees, P., & Temple, J. (2022). Methods for small area population forecasts: State-of-the-art and research needs. *Population Research and Policy Review*, 41(3), 865-898. https://doi.org/10.1007/s11113-021-09671-6

---

## Glossary of Terms

**ACS (American Community Survey)**: An ongoing survey by the U.S. Census Bureau that provides detailed demographic, social, economic, and housing data annually. The 5-year estimates provide reliable data for areas with populations as small as 65,000.

**AMI (Area Median Income)**: The midpoint of a region's income distribution, meaning that half of households earn more and half earn less. Used by HUD to set income eligibility thresholds for housing programs.

**ARIMA (Autoregressive Integrated Moving Average)**: A statistical model for time series forecasting that combines autoregression, differencing, and moving averages. ARIMA(0,1,1) uses first-order differencing and a moving average term.

**CBSA (Core Based Statistical Area)**: A geographic area consisting of one or more counties anchored by an urban center of at least 10,000 population plus adjacent counties with strong commuting ties.

**CCD (Cohort Change Difference)**: An additive measure of population change between age cohorts: CCD = Pop(age i+5, year t+5) - Pop(age i, year t).

**CCR (Cohort Change Ratio)**: A multiplicative measure of population change between age cohorts: CCR = Pop(age i+5, year t+5) / Pop(age i, year t).

**Cohort-Component Method**: A demographic projection technique that divides population into age-sex cohorts and projects each group forward using fertility, mortality, and migration assumptions.

**FRED (Federal Reserve Economic Data)**: An online database of economic data maintained by the Federal Reserve Bank of St. Louis, including interest rates, employment, and economic indicators.

**GEOID**: A unique geographic identifier code used by the U.S. Census Bureau to identify specific geographic areas (states, counties, census tracts, etc.).

**HUD (U.S. Department of Housing and Urban Development)**: Federal agency responsible for housing policy, including setting Area Median Income standards for affordable housing programs.

**Leslie Matrix**: A matrix representation of age-structured population projection where the first row contains fertility rates and the subdiagonal contains survival/migration rates.

**MOE (Margin of Error)**: A measure of uncertainty in ACS estimates, representing the 90% confidence interval around the point estimate.

**Overcrowding**: Housing condition where there is more than 1.0 person per room, excluding bathrooms, kitchens, and hallways.

**PCHIP (Piecewise Cubic Hermite Interpolating Polynomial)**: An interpolation method that produces smooth curves while preserving monotonicity and avoiding oscillations.

**PMI (Private Mortgage Insurance)**: Insurance required by lenders when down payment is less than 20% of home value, typically 0.5% of loan amount annually.

**PostGIS**: A spatial database extension for PostgreSQL that enables storage and analysis of geographic data.

**PUMA (Public Use Microdata Area)**: A geographic area containing at least 100,000 people, used for releasing census microdata while protecting privacy.

**PUMS (Public Use Microdata Sample)**: Individual-level Census data with geographic identifiers no smaller than PUMAs, allowing custom tabulations.

**Substandard Housing**: Housing units lacking complete plumbing facilities (hot/cold running water, flush toilet, bathtub/shower) or complete kitchen facilities (sink with running water, stove/range, refrigerator).

**TIGER (Topologically Integrated Geographic Encoding and Referencing)**: The U.S. Census Bureau's geographic database that provides boundary files for all census geographies.

**Vacancy Rate**: Percentage of housing units that are vacant, calculated as vacant units divided by total housing units.

**ZCTA (ZIP Code Tabulation Area)**: A generalized areal representation of U.S. Postal Service ZIP Code service areas, created by the Census Bureau for statistical purposes.
