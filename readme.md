# **MBTA Metric Dictionary**
 
This dictionary describes the methodology and source for each bus performance metric available in the MBTA Bus Visualization Tool. 

## Contents
- [Definitions](#definitions) 
- [Data Preparation and Cleaning](#data-preparation-and-cleaning) 
- [Metric List](#metric-list) 

## **Definitions:**

- **APC**: Automatic Passenger Counting. A system that collects and stores passenger boardings and alightings at each stop.
- **AVL**: Automatic Vehicle Location. A system that collects and stores bus location data at specified time intervals or events. 
- **Corridor-level metrics**: Metrics that are calculated for a segment where there are multiple overlapping routes. Corrdidor-level metrics involve aggregation of data for all routes serving the corridor in each direction.
- **Data field**: The schemas, tables and columns of the data used in the metric calculations within the MBTA Research database. 
- **Data source**: The general source of the data used for the metric calculations (some combination of APC, AVL, ODX and GTFS). 
- **GTFS**: General Transit Feed Specification. A data specification for representing public transit schedules and geometric information in a tabular format. Used as a data source for metrics that involves schedule information or distance between stops
- **Min. Sample**: The smallest period over which data is aggregated for each metric. This limit is applied in order to provide a reasonable degree of accuracy and relevance for metric calculations. Typically the minimum sample is one month due to daily variability. If the metrics involve ODX or APC data the minimum sample is increased to three months due to incomplete coverage (APC) or inherent uncertainty (ODX).
- **Periods**: Metrics can be visualized for a range of periods. Most are currently avaialble for average weekday at the hourly level.
- **Route-level metrics**: Metrics that are aggregated at the route level for each unique route ID. If there are multiple variants of a route, the route-level metrics are calculated separately.
- **Route Variant**: Multiple patterns of stops that use the same route ID. For example, there may be two variants of a route that end at different stops.
- **Segment-level metrics**: Routes are divided into segments for higher resolution analysis. A segment is a section of a route that spans between two consecutive stops. Segment-level metrics are calculated separately for each route in each direction. If there are multiple variants of the same route serving a segment, they will be treated as they same route for the purposes of calculating  segment-level metrics.

## **Data Preparation and Cleaning**

All data needed for MBTA bus metric calculation is stored on the private MBTA Research server in the mbta_dw database.

Tables in the apc schema contain both passenger loading and timestamp information for each stop event. APC is used over AVL for observed stop times because the APC tables include dwell time at each stop. This enables the calculation of speed metrics with and without dwell time included. A “minimum sampling period” is defined for each metric to ensure that there is sufficient data to produce a meaningful average even if a small number busses do not have APC systems installed. See the definitions section below for more information on minimum sampling times and metric calculations.

Metrics are calculated using a general program that requires a standardized input table. Custom queries are used to collect and reformat MBTA data from the MBTA Research server to match the standard input table specifications. The query scripts can be found [here](https://github.mit.edu/caros/MBTA-bus-visualization-tool/tree/master/queries). The standard input table includes the following columns:

| Stop Time | Route ID | Stop ID | Stop Sequence | Distance Travelled | Dwell Time | Passenger Load | Passenger Board | Passenger Alight | Seated Capacity | Trip ID | 
| --- | --- | --- | --- | --- | --- | --- | --- | --- |

Notes: The passenger loading is passenger loading upon departure from the stop. Route ID, Stop ID, Trip ID and Stop Sequence must match their corresponding GTFS fields. 

This information can be collected almost entirely from the apc schema. The apc.stop table contains records for every stop event recorded by the APC system and is generated by the MBTA with a lag of approximately 6 weeks from the date of collection. Data for the following input table columns are pulled directly from the apc.stop table:

Input Table Column Name | Database Column Name
| --- | --- |
Stop Time | apc.stop.actstoptime
Stop ID | apc.stop.stopid
Stop Sequence | apc.stop.stopseqid
Distance Travelled | apc.stop.actmilessincelaststop
Dwell Time | apc.stop.dwelltotmins
Passenger Load | apc.stop.psgrload
Passenger Board | apc.stop.psgron
Passenger Alight | apc.stop.psgroff

The Route ID and Trip ID for each stop event must be converted from the APC ID (recorded as apc.stop.route and apc.stop.trip) to the GTFS ID using lookup tables:

1.	The Route ID lookup table is load.loadprofileroutes, where there is a GTFS Route ID in load.loadprofileroutes.route that corresponds with each APC Route ID in load.loadprofileroutes.apcroute. 

2.	Matching Trip IDs is more convoluted than routes. The Route ID, Direction, Trip Date, Vehicle ID and Scheduled Start Time (encoded in apc.stop.trip) from the APC record are used to match an AVL record in the avl.trippiece table and get the corresponding Hastus Trip ID. 

Similarly, the seated capacity of the vehicle is found using a lookup table matching the bus id in apc.stop.bus to the corresponding bin in load.bus_vehicle.nseats. 

The table is sorted by date, Trip ID and Stop Sequence so that each stop event is shown sequentially by trip. The range of dates for the sample is entered manually in the metric calculation program. 

There are three issues that arise as a result of the way this APC data is generated:

- First, there are records for stop events at stops that are skipped. The stop times are estimates, but in the MBTA’s case the timestamps appear to be relatively accurate so they are used for metric calculations. This is not necessarily true for other agencies.

- The second issue is that the passenger counting sensor for APC systems are not perfectly accurate, so it is possible to end up with an unbalanced set of boardings and alightings on a trip. These records are post-processed by the MBTA's vendor software to balance the loading. Any non-zero load is proportionally distributed in whole numbers at other stops on the trip such that the final load is zero. The system assumes all imbalances are caused by missed boardings/alightings, so boardings are added if the net trip flow is negative and alightings are added if the net flow is positive. 

- Third, there are APC records for non-revenue trips such as relocating busses between depots. Conveniently, these records always have apc.stop.route = 0, so any records meeting that criterion are dropped. 


## **Metric List**

|1.|Stop Spacing|
|:---------------|:-----------------| 
|Definition|Distance between stops.|
|Unit|Miles|
|Data source|GTFS + OpenStreetMap|
|Data field|gtfs.shapes.shape\_pt\_lat, gtfs.shapes.shape\_pt\_lon |
|Periods|N/A (Not period-dependent)|
|Min. Sample|One Month|
|Segment-level description|Sum of the distance of each road section between the two stops of the segment|
|Corridor-level description|N/A.
|Route-level description|Total route distance divided by the total number of stops on the route minus one|
|Notes|The actual coordinates used for road network distance calculations are the outputs from matching the GTFS coordinates to the Open Street Map road network using the Valhalla map-matching library. (https://github.com/valhalla/valhalla)|

|2.|Scheduled Frequency|
|:--|:--| 
|Definition|Number of scheduled buses per hour.|
|Unit|Buses per Hour|
|Data source|GTFS|
|Data field|gtfs.stop\_times.arrival\_time|
|Periods|Weekday Average by Hour|
|Min. Sample|One month|
|Segment-level description|The number of scheduled bus arrivals at the first stop of a segment during a period, divided by the number of hours in the period.|
|Corridor-level description|The total number of scheduled bus arrivals across all routes at the first stop of a corridor during a period, divided by the number of hours in the period.|
|Route-level description|The number of scheduled bus arrivals at the first stop of the route during a period, divided by the number of hours in the period.|
|Notes| |

|3.|Observed Frequency|
|:--|:--| 
|Definition| Number of observed buses per hour.|
|Unit| Buses per Hour|
|Data source| APC |
|Data field| apc.stop.actstoptime|
|Periods|Weekday Average by Hour|
|Min. Sample|One month|
|Segment-level description| The number of observed bus arrivals at the first stop of a segment during a period, divided by the number of hours in the period.|
|Corridor-level description| The total number of observed bus arrivals across all routes at the first stop of a corridor during a period, divided by the number of hours in the period.|
|Route-level description| The number of observed bus arrivals at the first stop of the route during a period, divided by the number of hours in the period.|
|Notes| |
    
|4. |Running Time |
|:--- |:--- | 
| Definition | Observed travel time between two stops.|
|Unit | Minutes|
|Data source | APC |
|Data field| apc.stop.actstoptime|
|Periods|Weekday Average by Hour|
|Min. Sample | One month|
|Segment-level description | The average time elapsed between bus arrivals at the first stop of the segment and the second stop of the segment. For the first segment in each route, running time will be calculated by taking the elapsed time between departure from the first stop and arrival at the second stop.|
|Corridor-level description | The average time elapsed between bus arrivals at the first stop of the corridor and the second stop of the corridor across all routes serving the corridor. If the corridor is the first segment in a route, running time will be calculated by taking the elapsed time between departure from the first stop and arrival at the second stop.|
|Route-level description | The average time elapsed between bus departure from the first stop of the route and bus arrival at the last stop of the route.|
|Notes | |

|5.|Scheduled Speed|
|:--- |:--- | 
|Definition | Scheduled travel speed between two points.|
|Unit | Miles per Hour|
|Data source | GTFS|
|Data field | gtfs.stop\_times.arrival\_time|
|Periods|Weekday Average by Hour|
|Min. Sample | One month|
|Segment-level description | The distance between the two stops of the segment divided by the average time scheduled between bus arrivals at the first stop of the segment and the second stop of the segment. For the first segment in each route, the scheduled travel time will be calculated by taking the scheduled time between departure from the first stop and arrival at the second stop.|
|Corridor-level description | The distance between the two stops of the corridor divided by the average scheduled time between bus arrivals at the first stop of the corridor and the second stop of the corridor across all routes serving the corridor. If the corridor is the first segment in a route, scheduled travel time will be calculated by taking the scheduled time between departure from the first stop and arrival at the second stop.|
|Route-level description | The total route distance divided by the average time scheduled between bus departure from the first stop of the route and bus arrival at the last stop of the route.|
|Notes | See **1. Stop Spacing** for details of the distance calculations. Because stops can be very close together, this metric may show very high or low scheduled speed if the GTFS arrival times are rounded.|
    
|6.|Observed Speed with Dwell |
|:--- |:--- | 
|Definition | Observed travel speed between two stops including dwell time at the first stop.|
|Unit | Miles per Hour|
|Data source | APC |
|Data field| apc.stop.actstoptime|
|Periods|Weekday Average by Hour|
|Min. Sample | One month|
|Segment-level description | The distance between the two stops of the segment divided by the average running time.|
|Corridor-level description | The distance between the two stops of the corridor divided by the corridor running time.|
|Route-level description | The total route distance divided by the route running time.|
|Notes | See **1. Stop Spacing** for details of the distance calculations and **4. Running Time** for details of the running time calculations.|

|7.|Observed Speed without Dwell |
|:--- |:--- | 
|Definition | Observed travel speed between two points with dwell time removed.|
|Unit | Miles per Hour|
|Data source | APC |
|Data field| apc.stop.actstoptime|
|Periods|Weekday Average by Hour|
|Min. Sample | One month|
|Segment-level description | The distance between the two stops of the segment divided by the average running time with dwell time subtracted.|
|Corridor-level description | The distance between the two stops of the corridor divided by the corridor running time with dwell time subtracted.|
|Route-level description | The total route distance divided by the route running time with all dwell times subtracted.|
|Notes | See **1. Stop Spacing** for details of the distance calculations and **4. Running Time** for details of the running time calculations.|


|8.|Occupancy |
|:--- |:--- | 
|Definition | Number of passengers on a bus at a given location.|
|Unit | Passengers|
|Data source | APC or ODX|
|Data field | apc.stop.psgrload or load.bus\_load.flow\_scaled\_apc|
|Periods|Weekday Average by Hour|
|Min. Sample | Three months|
|Segment-level description | The average number of passengers on board each bus after the bus has departed the first stop of the segment across the study period.|
|Corridor-level description | The number of passengers on board each bus after the bus has departed the first stop of the corridor, averaged across the study period for all routes that serve the corridor.|
|Route-level description | The average number of passengers on board each bus for each segment of the route.|
|Notes | |
    
|9.|Arrival Delay|
|:--- |:--- | 
|Definition | Difference between the scheduled bus arrival time and the actual bus arrival time.|
|Unit | Seconds|
|Data source | GTFS + APC |
|Data field | gtfs.stop\_times.arrival\_time, apc.stop.actstoptime|
|Periods|Weekday Average by Hour|
|Min. Sample | One month|
|Segment-level description | The average of the observed arrival time minus the scheduled arrival time at the first stop of the segment.|
|Corridor-level description | The average of the observed arrival time minus the scheduled arrival time at the first stop of the corridor across all routes serving the corridor.|
|Route-level description | The average delay across all segments in the route.|
|Notes | If the actual arrival time precedes the scheduled arrival time, the arrival delay is considered to be zero seconds. If there is no observed arrival time for a given trip, that trip is excluded from the average calculation.|

|10.|Expected Wait Time|
|:--- |:--- | 
|Definition | Expected passenger wait time based on actual headway distribution.|
|Unit | Minutes|
|Data source | APC |
|Data field |apc.stop.actstoptime|
|Periods|Weekday Average by Hour|
|Min. Sample | One month|
|Segment-level description | The expected wait time for passengers assuming a uniform passenger arrival time. The calculation is based on the mean and standard deviation of the headway, determined by the difference between arrival times of consecutive buses at the first stop of the segment throughout the time period. Expected wait time is given by *E(w) = m/2 + s^2 / (2m)*, where *m* is the headway mean and *s* is the headway standard deviation.|
|Corridor-level description | Same as the segment-level metric, but the combined headway for all routes serving the corridor is used.|
|Route-level description | Average of the expected wait time for each segment in the route.|
|Notes | The segment-level and corridor-level metric calculations will return a null value unless there are two or more arrivals during the time period.|

|11.|Excess Wait Time|
|:--- |:--- | 
|Definition | Difference between the expected wait time generated from the scheduled and observed headway distributions.|
|Unit | Minutes|
|Data source | GTFS + APC|
|Data fields | gtfs.stop\_times.arrival\_time, apc.stop.actstoptime|
|Periods|Weekday Average by Hour|
|Min. Sample | One month|
|Segment-level description | The expected passenger wait time when observed headways are used minus the expected passenger wait time when scheduled headways are used. This is averaged across the time period for headways at the first stop in the segment.|
|Corridor-level description | Same as the segment-level metric, but the combined headway for all routes serving the corridor is used.|
|Route-level description | Average of the excess wait time at each segment in the route.|
|Notes | See **9. Expected Wait Time** for the expected travel time formula. If the observed expected passenger wait time is less than the scheduled expected passenger wait time, the excess wait time is considered to be zero minutes.|

|12.|Excess Travel Time |
|:--- |:--- | 
|Definition | Difference between actual travel time and scheduled travel time.|
|Unit | Minutes|
|Data source | GTFS + APC|
|Data fields | gtfs.stop\_times.arrival\_time, apc.stop.actstoptime|
|Periods|Weekday Average by Hour|
|Min. Sample | One month|
|Segment-level description | Travel time is calculated as the difference between arrival times at the two stops of the segment. Excess travel time is the observed travel time minus the scheduled travel time, averaged across the time period.|
|Corridor-level description | The observed travel time minus the scheduled travel time, averaged across the time period for all routes serving the corridor.|
|Route-level description | The observed travel time minus the scheduled travel time between the first and last stops of the route. This is averaged across all trips in the time period.|
|Notes | Negative excess travel time indicates that the scheduled travel time is slower than the actual travel time.|

|13.|Crowding |
|:--- |:--- | 
|Definition | Passenger load as a percentage of seated capacity|
|Unit | Percent|
|Data source | APC + Bus Capacity Lookup|
|Data fields | apc.stop.actstoptime, load.bus_vehicle.nseats|
|Periods | Weekday Average by Hour|
|Min. Sample | One month|
|Segment-level description | Crowding is calculated by dividing the passenger load between two stops by the fixed number of seats on the vehicle. |
|Corridor-level description | Crowding levels are averaged across all routes serving the corridor. |
|Route-level description | Crowding levels are averaged across all segments in the route. |
|Notes | Crowding may exceed 100\% if there are more passengers than seats. |

|14.|Boardings |
|:--- |:--- | 
|Definition | Number of passengers that board the bus |
|Unit | Passengers per Trip |
|Data source | APC |
|Data field | apc.stop.psgron |
|Periods | Weekday Average by Hour|
|Min. Sample | One month|
|Segment-level description | Boardings is the average number of passengers that board the bus at the first stop in the segment across all trips in the period |
|Corridor-level description | Crowding levels are averaged across all routes serving the corridor. |
|Route-level description | Crowding levels are averaged across all segments in the route. |
|Notes | |

|15.|Revenue Hours |
|:--- |:--- | 
|Definition | Total duration of revenue vehicle service across all trips. |
|Unit | Hours |
|Data source | APC |
|Data field | apc.stop.actstoptime |
|Periods | Weekday Average by Hour |
|Min. Sample | One month |
|Segment-level description | Not available at the segment level. |
|Corridor-level description | Not available at the segment level. |
|Route-level description | First, the sum of the duration of revenue service is calculated on each day across all trips serving the route. Then the daily average is taken across all of the days in the time period. |
|Notes | |

|16.|Productivity |
|:--- |:--- | 
|Definition | Ridership per revenue hour. |
|Unit | Passengers per Hour |
|Data source | APC |
|Data fields | apc.stop.psgrload, apc.stop.actstoptime |
|Periods | Weekday Average by Hour |
|Min. Sample | One month |
|Segment-level description | Not available at the segment level. |
|Corridor-level description | Not available at the segment level. |
|Route-level description | The total ridership for the route divided by revenue hours. |
|Notes | See **15. Revenue Hours** for the revenue hour calculation methodology. |
