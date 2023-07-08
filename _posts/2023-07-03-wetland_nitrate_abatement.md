---
layout: post
title: "Modeling Nitrate Abatement from Wetlands" 
---

# Nitrate Removal From Wetlands

This was my final project from Environmental Engineering. The course focused on physics- and chemistry- based models with some biological components, 
and this was a great opportunity to integrate those works with the modeling approaches I learned in Groundwater Flow modeling, at about the same time. 

My goal with this project was to estimate, to a first approximation, the amount of nitrate abated by wetlands and buffer strips around streams, using 
mathematical tools from groundwater, surface water, chemistry, and biological domains. I modeled processing time variations based on temperature, and 
used probability analysis in python to scale wetland extent to the size of relevant flood pulses. 

This project came partially from my work with the Geography Department at IU, where I was working with [Dr. Sally Letsinger](https://geography.indiana.edu/about/faculty/letsinger-sally.html) on a water budgeting study for SE Indiana.
I developed both stream-density and land-use maps from the project's geodatabase to inform my analysis, and coupled them with [stream gauge data](https://dashboard.waterdata.usgs.gov/app/nwd/en/?region=lower48&aoi=default) from
the USGS. 

![Land Use Map of the Driftwood Watershed](/Assets/Crop_Map_1.jpg)
![Stream Network Map of the Driftwood Watershed](/Assets/Watershed_map_1.jpg)

My visual style decisions for this mapping project were based upon expediency; this project was completed in three weeks, and GIS was only an initial part of the analytical
procedure. I tried to keep it simple while remaining understandable. 

I built my probabilistic hydrologic model on the empirical distribution derived from the USGS's period of record. As usual, I used Pandas for my 
aggregation and processing, which included aggregating from daily to three-day flows, performing ranking, and slicing for target (95% recurrence interval) flows. 
My focus was on encompassing some sense of likely extreme values for the distribution, based on [literature which indicates the majority of nitrate flows](https://pubs.acs.org/doi/10.1021/es052573n) occur 
during extreme events. This was also tied into the mass balance for nitrogen, which was estimated using NOAA's [SPARROW](https://www.usgs.gov/mission-areas/water-resources/science/sparrow-modeling-estimating-nutrient-sediment-and-dissolved) model.
My assumption was that 56% of all nitrate transport occurred at flows ≥ the 90th percentile, and simplified further by assuming that flows of 95th percentile carried all of the nitrogen above the 90th percentile. 

A constructed wetland can be modeled as a temperature-dependent plug flow reactor. In fact, temperature is an exceptionally important variable to consider when designing
a wetland for nutrient goals. Temperature and residence time are generally the limiting factors for nutrient processing in constructed wetlands, but the storms which are 
likely to carry nutrients tend to happen when temperature is sub-maximal (i.e. in the winter and spring) thus becoming the most important factors. To illustrate this issue, 
I simulated 1000 years of weather based on data from the watershed ([NOAA NCEI](https://www.ncei.noaa.gov/)). I assumed that recent weather was a decent approximator for the 
distribution of future weather and that each day's weather was an independent normally-distributed random variable. If I were to extend this project, this is an area I would focus on
for additional work, since independence doesn't really hold for weather time series data. I would also use a more performant generation technique to add a lot more data to the 
simulation, which would allow a much smoother final envelope. 

![Simulated Temperature Data](/Assets/Temp_Sim_Dist.png)

The simulated temperature data was then used to generate a distribution of required residence times and storage volumes for the constructed wetland to handle the nitrate 
emitted from my design event. 

![Nitrate Abatement Distribution](/Assets/Nitrate_Abatement_distribution.png)

While the bulk of acreages and residence times fall below 10000 acres/3 days, the rightward tail is still a substantial part of the distribution. My analytical heuristic was to reject any area over 10,000 acres as
infeasible, because of the financial and resource cost to construct such a wetland. Wetlands of about 10,000 acres [exist in Indiana](https://www.in.gov/dnr/fish-and-wildlife/properties/goose-pond-fwa/) but are rare. 
The right tail of this distribution describes all of the winter/spring/fall storms that occur closer to freezing. 

## Buffer Strip Model

My buffer strip model was derived from the same nitrate reduction equations used in the plug flow reactor above, coupled with a transport function from 1-d groundwater flow. 
This was a great opportunity for me to practice model development, and let me use skills from two areas to achieve a more sophisticated approximation (emphasis on the approximation)
of the nutrient processing capabilities of subsurface flow. 

The model is based on the equation for 1-d groundwater flow because I assumed that within a certain radius of a stream, all groundwater flow was unidirectional and towards the stream. By further assuming 
that riparian edges would be uniform, and neglecting any hyporheic exchange, I could model the buffer's capacity as the sum of a series of small slices. 

![Visual Representation of 1-d Groundwater Flow Model](/Assets/gw_flow_screenshot.png)

I coupled the equation for groundwater flow with that of a plug flow reactor to determine the length needed to process the total mass of nitrate from the design flow. This was a really rewarding derivation despite the 
simplifications I made to get to an answer in a reasonably short amount of time. I later learned that my assumption that nitrate was processed in the very shallow subsurface is wrong; really groundwater needs to infiltrate
deeper than I was modeling in order to start denitrifying, which limits processing capacity below what I predicted. I additionally excluded infiltration dynamics, which are important to consider in a less idealized system, 
especially with nonconductive soils like those typically found in southern Indiana. 

## Conclusions

This project was a very rewarding one, as it allowed me to practice developing modeling to answer design questions semi-independently. MI also found the process of documenting behavior in the face of uncertainty to be 
highly rewarding, and allowed me to explore the dimensions of the problem in a more naunced way. This project also let me think through the practical challenges to processsing nitrate in polluted agricultural wetlands, 
and changed my views on the utility of wetland restoration for nutrient pollution abatement, helping me understand its limitations in the real world.  



