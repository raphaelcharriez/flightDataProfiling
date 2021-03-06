# flightDataProfiling
Estimating Loading Factor with ADS-B flight position data 

## 1/ Hyothesis
Hypothesis 1: The higher is the better, you’ll get less air drag at higher altitude. The Lighter the aircraft, the higher it can go, and it most cases it will try to reach higher cruising altitude.

Hypothesis 2: A big part of the aircraft weight is driven by it's load factor (how many passengers and cargo it contains). Some figures: 

* A 737-800 has ~ 160 seats. 
  * each passenger + their luggages will weight ~ 90kg
  * That’s 14,400 kg for a full aircraft vs 0 kg for an empty aircraft
  
  * It’s a significant part of the weight of the aircraft (25% of max weight)
* For each passenger you’ll add extra fuel as well so that emplifies the variation.

Conclusion: By looking at the flight profile of an A/C we will be able to infer it's weight, which will be correlated to the number of passengers/cargo which should be correlated with the profitability of the flight for instance.

# 2/ Implementation, data gathering and cleaning 

Let's get 2Years worth of data from https://opensky-network.org/
* We track the position of every aircraft in the world minutes by minutes, in parquet format 2Years worth of data weights 3TB.
* Processing Stack : Spark.
* Priliminary treatment :
  ** Enrich with aircraft metadata (operator, type of aircraft ect) 
  ** Enrich with flight data (using open flight scheduling data joined on aircraft msn + time with the position recorded time
  
# 3/ Analytics

* Let's focus on the cruise phase
![A flight](https://github.com/raphaelcharriez/flightDataProfiling/blob/master/flight.png)
* We can see different flight phases, taxi out, take off, climbing, cruising, descent, landing, taxi in.
* For our purpose let's just isolate the most stable phase of the flight: the cruise phase 

```
### With SPARK:  ### 

from pyrspark.sql import functions as F

stable_vertical_speed = (F.col('infered_vertical_speed') < 300) & (F.col('infered_vertical_speed') > -420)
sufficient_altitude = (F.col('altitude_feet') > 25000)

series = series.withColumn(
       'cruise_point',
       F.whem(stable_vertical_speed & sufficient_altitude, 1).otherwise(0)
)

timestamp_cruise = series.groupBy(F.col('flight_id')).agg(
       F.min(F.when(F.col('cruise_point')==1, F.col('timestamp'))).alias('start_cruise'),
       F.max(F.when(F.col('cruise_point')==1, F.col('timestamp'))).alias('end_cruise')
)
series = series.join(timestamp_cruise, on=['flight_id'], how='left')

```

* Now we have enough data to compute a preliminary statistics: 
 ** Average altitude during cruise phase
 ** Average altitude during the early cruise phase (when weight of the aircraft should be maximum because no fuel burn yet) 

# 4/ Results

The Corona Virus crisis was a good time to test the index, since airline have been publlicly flying near empty aircrafts.  Let's see if we can capture that with our index.

![Early Result](https://github.com/raphaelcharriez/flightDataProfiling/blob/master/Screen%20Shot%202020-03-31%20at%209.05.11%20PM.png?raw=true)

Very Encouraging Early Results, we can see the average early cruise altitude increasing by thousands of feet as aircrafts fly empty. 

Next Step: train historical Data ny airline and compare with Airline's financial reporting.
