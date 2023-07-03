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

<p align="center">![Land Use Map of the Driftwood Watershed](/Assets/Crop_Map_1.jpg)</p>
<p align="center">![Stream Network Map of the Driftwood Watershed](/Assets/Watershed_map_1.jpg)</p>

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

<p align="center">![Simulated Temperature Data](/Assets/Temp_Sim_Dist.png)</p>

The simulated temperature data was then used to generate a distribution of required residence times and storage volumes for the constructed wetland to handle the nitrate 
emitted from my design event. 

<p align="center">![Nitrate Abatement Distribution](/Assets/Nitrate_Abatement_distribution.png)</p>

