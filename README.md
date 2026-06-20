# Wildfire Risk System

> work in progress - started June 2026

## Overview

Most publicly accessible wildfire tools tell you where fires are. What they don't do well is tell you which fires pose the greater risk, especially when there are multiple active clusters in a region at the same time. This project is an attempt at building a near-real-time risk scoring system for wildfires in the US, driven entirely by satellite data and open public sources. I also just like space and space-related stuff so this seemed like a cool and fun project for myself.

The system ingests satellite hotspot detections from NASA FIRMS, clusters them into fire events using spatiotemporal methods, and scores each event with a trained ML risk model. Predictions are served through a REST API.

Idea for the final product is to be a live service in which a user looks at a map of the US and see the hotspots for all current active fires, much like many fire-watching products now. The user can then search or zoom into a region, see the clustering of the wild fires, and have this region scored (with some maximum size of the region set). The end result is each of the displayed clusters in the region showing a rank/score, telling the user which active fires pose the greater risk. That is a v1 end goal, which once completed, I plan to add richer predictive information, such as directions in which the fire is going to spread, incorporating infrastructure data into the prediction (e.g. a residential area should be ranked higher than an open field), and so on.

## Folder Structure

- `prelim_eda/` - preliminary EDA on the data sources and APIs
- `planning/` - todos, architecture, and design decision reasoning


## Data Sources

- **[NASA FIRMS](https://firms.modaps.eosdis.nasa.gov/)** - satellite-based distribution of near-real-time active fire and thermal anomaly detections. Using VIIRS SNPP and NOAA-20. This is the primary data source, it tells us where the fires are.
- **[WFIGS](https://data-nifc.opendata.arcgis.com/)** - the authoritative source for current and historical wildfire perimeters in the US. This tells us the shape and size of a fire.
- **[ICS-209-PLUS](https://research.fs.usda.gov/firelab/products/dataandtools/ics-209-plus)** - a research-compiled dataset of incident reports for fire events. Used to generate ground truth labels for the risk model.
- **[HRRR](https://rapidrefresh.noaa.gov/hrrr/)** - high spatial resolution, hourly updated weather data across the US. Used to enrich features for the risk model.
