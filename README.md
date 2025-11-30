# PRC2025_DataChallenge_Team_unique-quicksand_PUBLIC
This is the team "unique-quicksand" public GitHub repository as submission for the PRC Data Challenge 2025 (https://ansperformance.eu/study/data-challenge/dc2025/).

# References
The following references have been considered to get inspirations on how to tackle the challenge:
- [ICAO Aircraft Engine Emissions Databank](https://www.easa.europa.eu/en/domains/environment/icao-aircraft-engine-emissions-databank)
providing for different engine types:
  - Fuel flow (_kg/sec_) at take off condition,
  - Fuel flow (_kg/sec_) at climb out condition,
  - Fuel flow (_kg/sec_) at approach condition,
  - Fuel flow (_kg/sec_) at idle condition.
- ["Fuel Consumption of the 50 Most Used Passenger Aircraft" by Marius Kühn, 2023-09-11](https://www.fzt.haw-hamburg.de/pers/Scholz/arbeiten/TextKuehn.pdf) 
providing and example of an A320 BADA Performance File (Revision 3.0, 1998) with fuel rate in _kg/min_.
- [Gabriel Jarry, Daniel Delahaye, Christophe Hurter. Towards aircraft generic Quick Access Recorder
fuel flow regression models for ADS-B data. International Conference on Research in Air Transportation, Eurocontrol FAA NTU, Jul 2024, Singapour, Singapore. ffhal-04636710](https://enac.hal.science/hal-04636710v1/document) 
and respective [Acropole](https://github.com/DGAC/Acropole) GitHub repository sounding similar to this challenge, but ultimately not used.
- ["Dealing with Outliers Using Three Robust Linear Regression Models" by Eryk Lewinson, 2022](https://towardsdatascience.com/dealing-with-outliers-using-three-robust-linear-regression-models-544cfbd00767/)

# Additional data set
Only the following additional data set is used:
- ["Aircraft Characteristics Database" by FAA](https://www.faa.gov/airports/engineering/aircraft_char_database) 
providing some global aircraft characteristics. 


# Step 1 - Read and explore PRC data
In this step we learned to read and join the provided data. 
Furthermore, we explored the data using statistical methods including histogram and box plots.

# Step 2 - Baseline model
The objective of our baseline model was
- to check we have in principle have understood the challenge and the provisioned data,
- to establish a coding framework in which later other models could be integrated and
- to get an understanding of the order of magnitude of the RMSE values on the ranking data.

## Implementation highlights
Following our short literature review and guided by engineering judgement, the foundation of our model is determining the 
**fuel burn rate** (= fuel flow rate) with unit of *kg/min* (i.e. _kilogram_ per _minute_) 
for each interval in the provided data:

```python
# Compute the duration of each interval
df_fuel['interval'] = df_fuel['end'] - df_fuel['start']
# Get duration of each interval in minutes (calculated from seconds)
df_fuel['interval_min'] = df_fuel['interval'].dt.total_seconds() / 60

# Compute the fuel burn rate as kg/min for each interval
df_fuel['fuel_rate'] = df_fuel['fuel_kg'] / df_fuel['interval_min']
```

Next, as **_baseline model_** the **median** of the ``fuel_rate`` per ``aircraft_type`` is computed over all training data intervals:
```python
# Group by 'aircraft_type' and compute the median of 'fuel_rate'
# .reset_index() converts the output back into a DataFrame
median_fuel_rate_df = df.groupby('aircraft_type', observed=True)['fuel_rate'].median().reset_index()
```
This **_baseline model_** is applied on all the intervals in the training and ranking data and gives the **estimated fuel burn rate** ``fuel_rate_est``:
```python
# Join estimated fuel_rate into the fuel DataFrame object
df_fuel = df_fuel.join(median_fuel_rate_df.set_index('aircraft_type'), rsuffix='_est', on='aircraft_type')
```
Finally, a "de-normalization" of the estimated fuel burn rate (``fuel_rate_est``) with interval 
duration (``interval_min``) is done to get the estimated fuel burn per interval in _kg_ (``fuel_kg_est``) for all 
intervals in the training and ranking data:
```python
df_fuel['fuel_kg_est'] = df_fuel['fuel_rate_est'] * df_fuel['interval_min']
```
The full Python Jupyter notebook code of the **_baseline model_** is given [here](https://github.com/olinbus/PRC2025_DataChallenge_Team_unique-quicksand_PUBLIC/blob/main/Step%202%20-%20Baseline%20Model%20cleaned.ipynb).

## Discussion 
The use of the **median** of the fuel burn rate per aircraft type as **_baseline model_** is motivated by assuming that:
- most flight intervals will be from **cruise** phases,
- the number of intervals during **climb** (higher fuel burn rate) and **descent** (lower fuel burn rate) phases are similar and roughly cancel each other out and
- the number of intervals during other flight phases is comparatively lower and again fuel burn rates partly cancel each other out as well and hence can be ignored for this model.

In addition, the **median** is used instead of the **mean** value, is it is less impacted by outliers.

How well this **_baseline model_** fits for cruise phase can be seen by comparing the median values per aircraft type against published data. 
For example the below table show the ``fuel_rate_est`` in _kg/min_ for selected narrow-body aircraft:

| ``aircraft_type`` | ``fuel_rate_est`` |
|-------------------|------|
| A320              | 39.9 | 
| A321              | 39.9 |
| A20N              | *18.0* |
| A21N              | 36.6 | 
| B737              | 36.9 |
| B738              | 39.9 | 
| B38M              | *24.3* |
| B39M              | 36.5 | 

Most of the above narrow-body aircraft have a fuel burn rate in the order of 40kg/min.
However, for ``A20N`` and ``B38M`` only about 20kg/min is estimated. By comparison to the other listed 
aircraft types and published fuel burn rates in cruise these estimates are obviously wrong. 

Closer inspection of the underlying data reveals these wrong estimates are not due to the simplicity 
of the "model". 
These wrong estimates rather highlight a very significant number of outliers in the ``fuel_kg`` 
interval data (assuming the ``start`` and ``end`` times can be trusted more). 
These outliers are observed also for other aircraft types.

In addition, during inspection of the ``fuel_kg`` interval data via histograms it is noticed 
that too many values are absolutely identical which is unexpected for measured data. A typical example 
of the observed issue is ``flight_id = "prc772539375"`` for ``A20N`` featuring expected 40kg/min as well 
as outlying 20kg/min intervals during cruise.
This raises the suspicion of these values in the data provided for this challenged being either model 
outputs or manipulated/corrupted in the organizers data pipeline.

Noting the mismatch highlighted for ``A20N`` and ``B38M`` (but not only restricted to these) being 
in the order of a factor 2 or 1/2=0.5  an issue concerning unit conversion between  _kg_ and _lb_ 
is suspected (conversion factor from _lb_ to _kg_ = 0.453592). 
However, no systematic of this issue could be identified and hence the suspicion remains an 
unproven hypothesis.

In order to test this "factor 2" hypothesis in our v1 submission the ``fuel_rate_est`` for ``A20N`` 
is doubled and applied to the ranking data (usage of flag ``bA20N_hack`` in our code). 
The RMSE (root-mean-squared-error) on the submitted ranking data increased from RMSE=266.5899 (v0) to 
RMSE=281.5008 (v1). Considering this, the conclusion is that not only the 
training data, but also the ranking data (and likely also the final data) suffer from the significant 
number of outliers.

In summary, this **_baseline model_** did meet our objectives of getting into this PRC 2025 challenge, 
entering the phase 1 ranking not on the last place and establishing a robust baseline for 
further true machine learning models.

# Further steps
In further steps we step-by-step and in several iterations replaced the above  **_baseline model_** 
with a [xgboost](https://xgboost.readthedocs.io/en/stable/) 
``XGBRegressor`` model with the following features:
- ``aircraft_type`` the provided aircraft type and to give some meaning to this also some aircraft characteristics
  - ``Num_Engines`` number of engine,
  - ``MTOW`` maximum take-off weight,
  - ``MLW`` maximum landing weight,
  - ``ac_L`` aircraft length and
  - ``ac_B2`` aircraft span squared.
- Some features giving the duration as well as absolute and relative timewise position of the interval in the flight.
- For the total flight from origin to destination: duration, distance and bearing. 
- From the provided track / position data at the middle of each interval: altitude, TAS, GS and ROC - where available 
and considering CAS to TAS conversion. 

Most of the steps turned out to improve the RMSE not only against the training data, but also on the ranking data.
However, RMSE improvements are only minor and a significant gap to the leading teams and even more towards the ground 
truth is recognized. We attribute the gaps towards the leading teams to our still rather simplistic model and not
getting our heads around to best cope with or even predict the outliers in training and ranking data.

As an attempt to deal with the outliers in the track / position data a [``RANSACRegressor``](https://scikit-learn.org/stable/modules/generated/sklearn.linear_model.RANSACRegressor.html) 
is used:
"_RANSAC is an iterative algorithm for the robust estimation of parameters from a subset of inliers from the complete data set._"

As the leading teams ranking submissions are so much better than our best XGBoost model, we refrain from 
publishing ours. Instead, we publish and apply our **_baseline model_** to the final data [here](https://github.com/olinbus/PRC2025_DataChallenge_Team_unique-quicksand_PUBLIC/blob/main/Step%202%20-%20Baseline%20Model%20cleaned.ipynb). 
With this we do not provide our "best", but our most robust estimate :)

# Conclusion
Again this PRC challenge was
- very professionally prepared: instructions, team provision, data, ranking, etc.,
- continuously and actively supported by the organizer and team,
- a true data and aviation challenge (luckily this year not a big data challenge),
- a great learning experience,
- bringing people across borders and various organizations together and
- last but not least: Great fun!

Congratulates to the winning team - very impressive performance (note: we are close to certain we will not be among the top teams). 
However, we are even more interested to understand how you made it!

Many thanks to the organizers for setting-up and running this challenge!

We are looking forward to any new challenge of this kind! Hoping to meet and challenge again at some point :)



