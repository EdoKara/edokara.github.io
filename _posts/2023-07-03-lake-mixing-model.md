---
layout: post
title: "Lake Mixing Model in Rust"
---

# Lake Mixing Model

This is a current side project to get some practice with rust coding and become more comfortable with lower-level programming in general.

The goal is to create a model which describes heat flow and mixing in lacustrine environments; ultimately this could be used to describe temperature variability and heat budgets for lakes. 
Rust appeals to me for this project because of its potential performance; this could enable either greater detail or greater ability to model ensembles. 

## Current State

I'm working on adding in solar radiative forcing, which starts with solar position and exposure-hours. To start with I'm using some [helpful equations from NOAA](https://gml.noaa.gov/grad/solcalc/calcdetails.html#:~:text=The%20calculations%20in%20the%20NOAA%20Sunrise%2FSunset%20and%20Solar,and%20within%2010%20minutes%20outside%20of%20those%20latitudes.) which describe the position of the sun based on julian day/century and the 
trigonometry of solar positions.

