# Ex-09: Time series analysis

## **Objective:**

-   Analyze AQI time series data to identify underlying patterns, trends, and seasonality.

-   Apply ARIMA models to forecast future AQI values.

-   Explore and apply detrending methods to examine the time series data without its trend component.

-   Investigate the seasonality in the data, understanding how AQI values change over different times of the year.

### **Prerequisites:**

-   **Software and Libraries:** Ensure Python, Jupyter Notebook, and necessary libraries (**`pandas`**, **`matplotlib`**, **`statsmodels`**, **`pmdarima`**) are installed.

-   **Datasets:** Access to the provided dataset with **`date`** and **`aqi_value`** columns, among others, to perform the analysis.

### **Key Concepts:**

#### Time Series Analysis

-   **Time Series Data:** Data points collected or recorded at specific time intervals.

-   **Trend:** The long-term movement in time series data, showing an increase or decrease in the data over time.

-   **Seasonality:** Regular patterns or cycles of fluctuations in time series data that occur due to seasonal factors.

#### ARIMA Modeling

-   [**ARIMA**](https://en.wikipedia.org/wiki/Autoregressive_integrated_moving_average) **(AutoRegressive Integrated Moving Average):** A popular statistical method for time series forecasting that captures different aspects of the data, including trend and seasonality.

-   **Parameters (p, d, q):**

    -   **`p`**: The number of lag observations included in the model (AR part).

    -   **`d`**: The degree of differencing required to make the time series stationary.

    -   **`q`**: The size of the moving average window (MA part).

#### Stationarity and Differencing

-   **Stationarity:** A characteristic of a time series whose statistical properties (mean, variance) do not change over time.

-   **Differencing:** A method of transforming a time series to make it stationary by subtracting the previous observation from the current observation.

#### Detrending

-   **Detrending:** The process of removing the trend component from a time series to analyze the cyclical and irregular components.

#### Seasonality Analysis

-   **Seasonal Decompose:** A method to separate out the seasonal component from the time series data, allowing for analysis of specific patterns that repeat over fixed periods.

## Dataset:

The data this week comes from the EPA's measurements on air quality for Tucson, AZ core-based statistical area (CBSA) for 2022.

We'll use the dataset: `ad_aqi_tracker_data-2023.csv`, which includes daily observations on air quality, along with multi-year averages.

### Metadata for `ad_aqi_tracker_data-2023.csv`**:**

| Variable             | Class     | Description                          |
|----------------------|-----------|--------------------------------------|
| **`Date`**           | DateTime  | Date of observation                  |
| `AQI Values`         | int       | Air quality index reading            |
| **`Main Pollutant`** | character | Primary pollutant at time of reading |
| **`Site Name`**      | character | Name of collection site              |
| `Site ID`            | character | ID of collection site                |
| **`Source`**         | double    | Data source                          |

***Note**: You will have to change the data type for some columns to match the above.*

(Source: <https://www.airnow.gov/aqi-basics>)

## **Question:**

How can we apply ARIMA modeling to forecast future Air Quality Index (AQI) values based on historical data, and what insights can be gained from detrending and analyzing the seasonality in AQI time series data?

## **Step 1: Setup and Data Preprocessing**

1.  Load the dataset into a pandas DataFrame.

2.  Convert the **`date`** column to datetime format and set it as the index of the DataFrame.

3.  Convert the `aqi_value` column as needed.

4.  Plot the **`aqi_value`** time series to visually inspect the data.

```{python}
import pandas as pd
import matplotlib.pyplot as plt
from skimpy import clean_columns

# Load the dataset
df = pd.read_csv('path_to_your_file.csv')

# Clean column names
df = clean_columns(df)

# Assign and remove NAs
df.replace({'.': np.nan, '': np.nan}, inplace = True)
df.dropna(inplace=True)

# Convert 'date' to datetime and set as index
df['date'] = pd.to_datetime(df['date'])
df.set_index('date', inplace = True)

# Plot the AQI values
df['aqi_value'].plot(title = 'AQI Time Series')
plt.ylabel('AQI Value')
plt.show()
```

## **Step 2: Time Series Decomposition**

1.  Use **`seasonal_decompose`** from the **`statsmodels`** package to decompose the time series into trend, seasonal, and residual components.

2.  Plot the decomposed components to understand the underlying patterns.

```{python}
from statsmodels.tsa.seasonal import seasonal_decompose

# Decompose the time series
decomposition = seasonal_decompose(df['aqi_value'], model = 'additive')

# Plot the decomposed components
decomposition.plot()
plt.show()
```

## **Part 3: Testing for Stationarity**

1.  Perform an [Augmented Dickey-Fuller](https://en.wikipedia.org/wiki/Augmented_Dickey%E2%80%93Fuller_test) (ADF) test to check the stationarity of the time series.

2.  If the series is not stationary, apply differencing to make it stationary.

```{python}
from statsmodels.tsa.stattools import adfuller

# Perform Augmented Dickey-Fuller test
result = adfuller(df['aqi_value'])
print('ADF Statistic: %f' % result[0])
print('p-value: %f' % result[1])

# Interpretation
if result[1] > 0.05:
    print("Series is not stationary")
else:
    print("Series is stationary")
```

## **Step 4: ARIMA Model**

1.  Use the **`auto_arima`** function from the **`pmdarima`** package to identify the optimal parameters (p,d,q) for the ARIMA model.

2.  Fit an ARIMA model with the identified parameters.

3.  Plot the original vs. fitted values to assess the model's performance.

```{python}
from pmdarima import auto_arima

# Identify the optimal ARIMA model
auto_model = auto_arima(df['aqi_value'], start_p = 1, start_q = 1,
                        test = 'adf',         # use adftest to find optimal 'd'
                        max_p = 3, max_q = 3, # maximum p and q
                        m = 1,                # frequency of series
                        d = None,             # let model determine 'd'
                        seasonal = False,     # No Seasonality
                        start_P = 0, 
                        D = 0, 
                        trace = True,
                        error_action = 'ignore',  
                        suppress_warnings = True, 
                        stepwise = True)

print(auto_model.summary())

# Fit ARIMA model
model = auto_model.fit(df['aqi_value'])

# Plot original vs fitted values
df['fitted'] = model.predict_in_sample()
df[['aqi_value', 'fitted']].plot(title='Original vs. Fitted Values')
plt.show()
```

## **Part 5: Forecasting**

1.  Forecast AQI values for the next 30 days using the fitted ARIMA model.

2.  Plot the forecasted values alongside the historical data to visualize the forecast.

```{python}
# Forecast the next 30 days
forecast, conf_int = model.predict(n_periods = 30, return_conf_int = True)

# Plot the forecast
plt.figure(figsize = (8, 6))
plt.plot(df.index, df['aqi_value'], label = 'Historical')
plt.plot(pd.date_range(df.index[-1], periods = 31, closed = 'right'), forecast, label='Forecast')
plt.fill_between(pd.date_range(df.index[-1], periods = 31, closed = 'right'), conf_int[:, 0], conf_int[:, 1], color = 'red', alpha = 0.3)
plt.title('AQI Forecast')
plt.legend()
plt.show()
```

## **Part 6: Detrending and Seasonality Analysis**

1.  Explore different detrending methods (e.g., subtracting a moving average, polynomial detrending) using the **`detrend_aqi`** and **`poly_trend`** columns.

2.  Analyze seasonality patterns in the detrended data.

```{python}
# Detrending using moving average
df['moving_avg'] = df['aqi_value'].rolling(window = 12).mean()
df['detrended'] = df['aqi_value'] - df['moving_avg']

# Plot detrended data
df[['detrended']].plot(title='Detrended AQI Time Series')
plt.show()

# Assuming seasonality was identified, you can further analyze it,
# for example, by averaging detrended values by month or another relevant period.
```

## **Submission:**

-   Submit your Jupyter Notebook via the course's learning management system, including your code, visualizations, and a brief discussion of your findings regarding the impact of cage-free practices on egg production.
