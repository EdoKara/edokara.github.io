---
layout: post
title: "Contributions to my Stream Ecology Capstone"
---

# Background

Stream Ecology was my capstone course for the graduate program at SPEA, and was a great way to put my ecology and data analysis skills to work on a real case study. 

The basic structure of the class was to learn about one aspect of an EPA rapid bioassessment
each week, practice sample collection and analysis for that portion in field/lab days, and then write about our findings in reports.

There were 3 reports over the course of the semester, and the final one brought together all of our findings into one comprehensive discussion of the ecological health of our target site.

Our group worked in Upper Stephens Creek, a 3nd-order wadeable stream near Bloomington, Indiana. The stream is downcut but still has decent structure despite sitting on bedrock. It is straight near the target reach but has cobbles and some hydrologic variety, especially some decent riffles.

![Picture of our target reach of Upper Stephens Creek](/Assets/SiteCharacterization4.jpg)

# Major Analysis Components

below are a few of the components I helped out with. 

## Flow & Temp/Cond time series

Flow was collected each week using a top-setting wading rod. This was a welcome upgrade from some of my previous work (especially my undergraduate work) which entailed a lot of work with a plastic, un-settable flow meter with unreliable connection. It was fun to use specialized equipment, and it even helped me stay upright on a few occasions during high flow conditions. 

It seemed to rain every Friday when we did our field work over the course of the semester, so we had decent characterization of the high flow events and the temperature/conductivity profile which came along with them. One of my favorite parts of our data collection campaign was the continuous monitoring we did using[HOBOWare dataloggers](https://www.onsetcomp.com). I wanted to analyze the periodicity of temperature and conductivity that we were able to collect from this data and got a pretty neat graph out of it in the end: 

![Temperature and Conductivity Trend Decomposition](/Assets/Temp_Decomp.png)

I was really happy with how this graph turned out; the color coding, sharing of time axes, and the positioning and subsetting work was all really satisfying, and really made me feel like I'm getting the hang of matplotlib. Trend decompositions are also a pretty neat statistical analysis to perform, and I think they're a really helpful way to analyze a time series like the one shown above. In this case, the mirrored periodicities of conductivity and temperature are due to the datalogger reporting in absolute instead of specific terms. While temperature has diel fluctuations which pretty much follow the weather (showing that the stream is responsive to variations in atmospheric temperature instead of having stable T) the conductivity has a pronounced drop on 3/4/2023. This signifies a dilution event, where the conductivity was reduced due to inputs from rainfall with a lower dissolved solid load. 

<!-- ## Sediment Transport

I did some analysis of the sediment characteristics of Upper Stephens Creek as well. -->


