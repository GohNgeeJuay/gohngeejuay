---
title: "Atlite with ERA5-Land"
date: 2026-02-13
author: "Jake Goh"
summary: "Using Atlite with higher resolution ERA5-Land dataset"
showToc: true
tocOpen: false
draft: false
---


## Introduction

In my previous blog post, I established why I was using ERA5-Land instead of the ERA5 dataset. This was because I wanted to get higher resolution in terms of PV potentials in Malaysia, which was lacking in existing datasets. The ERA5 dataset provides ~31 km grid resolution, which is already good, but I wanted to get even more granular data. Using the ERA5-Land dataset, it can provide detail up to ~9 km spatial resolution.

However, this posed a challenge since **Atlite requires solar irradiance inputs (especially Global Horizontal Irradiance, GHI, and the top-of-atmosphere solar radiation used for irradiance decomposition)** for PV calculation. ERA5-Land does not provide the same set of irradiance variables that Atlite’s ERA5 workflow expects (such as direct solar radiation at the surface). Because of this, I had to calculate the **Top of Atmosphere (TOA) solar influx** manually to enable Atlite’s PV pipeline to work correctly. I spent time analysing and validating that the calculated TOA influx behaves as expected and is physically reasonable.

This blog posts explains on a high-level what has been done to integrate the ERA5-Land dataset with Atlite, such that I can generate PV potential estimates for Malaysia using higher-resolution reanalysis data.

---

## Goals

1. **Integrate ERA5-Land into Atlite**
    
    - Context: Atlite already supports a few climate datasets such as ERA5, SARAH, and GEBCO, and provides a clean workflow for downloading, preparing, and converting climate data into renewable energy potential estimates.
        
    - To extend Atlite so that ERA5-Land becomes a supported dataset option. 
        
2. **Use Atlite to generate PV potential in Malaysia**
    
    - After integration, use Atlite to generate PV potential across Malaysia at ERA5-Land’s higher spatial resolution (~9 km).
        
    - To run this consistently across multiple years (2020–2025) to enable year-to-year comparison.
        
3. **Analyse and validate the PV results**
    
    - To compare ERA5-Land outputs against Atlite’s existing ERA5 pipeline to confirm that the higher-resolution data produces more granular spatial patterns.
        
    - To run sanity checks (ranges, spatial consistency, expected regional trends) to ensure the results are physically reasonable. 


---

## Design 

![Design flowchart](/notebooks/atlite_era5_land_analysis/Design_flowchart.png)

In this blog post, I will explain the Atlite integration using the previous data processing. Subsequently, use the newly supported dataset to make predictions of potential solar energy output.


## Procedure


### 1. Add ERA5-Land as a new dataset option in Atlite

I created a new dataset module specifically for ERA5-Land using Atlite’s existing ERA5 dataset module as a reference. I reused the same structure, functions and conventions to match the same interface as the ERA5 module. Atlite expects supported datasets to be able to implement certain functions such as download functions, data sanitisation functions and so on. 

The main goal is to ensure that my ERA5-Land module also implements the interface, and it behaves like another supported dataset inside Atlite. This means it should be able to download climate data, load it into Xarray, and return variables in the format Atlite expects.

&nbsp;

### 2. Make ERA5-Land consistent with Atlite’s internal conventions

ERA5-Land variables follow ECMWF/CDS naming conventions such as `fdir`, `ssrd`, and `tisr`. Atlite, however, expects consistent internal names across all datasets. I needed to rename variables, convert units where necessary, and ensure the coordinates and dimensions match Atlite’s standards. This step is important because most of Atlite’s PV pipeline assumes these names and formats exist.

&nbsp;

### 3. Check what the PV pipeline needs and fill in missing inputs

Atlite’s PV model needs two main types of climate inputs: **solar radiation** (to estimate how much sunlight reaches a solar panel) and **temperature** (to account for efficiency losses when panels heat up). ERA5-Land provides both **surface solar radiation downwards** (SSRD) and **2m temperature** (t2m), so at first glance it looks like we already have what we need.

However, Atlite needs to know not just how much sunlight reaches the ground in total, but also how much of it comes as **direct sun** versus **diffuse light** (sunlight scattered by clouds and the atmosphere). With the standard ERA5 dataset, this split is easy because the dataset already includes both the total sunlight at the surface and the direct portion, so the diffuse portion can be derived as “total minus direct”. ERA5-Land does not provide the direct component, so I had to compute the missing top-of-atmosphere solar input and let Atlite’s built-in model estimate the direct vs diffuse split from there. This was done in the previous analysis and shown in the previous blog article. 

With this calculation, Atlite can now run its PV pipeline end-to-end using ERA5-Land even without the extra radiation variables that ERA5 provides.

&nbsp;

### 4. Register ERA5-Land so it becomes selectable in Atlite

After the dataset module worked, I added it into Atlite’s dataset registry. Atlite keeps a record of all supported dataset in a file, and this step is what makes `Cutout(module="era5_land")` possible. Without this, the module exists in code but cannot be used through the normal Atlite workflow such as end-to-end PV estimation.

&nbsp;

### 5. Run an end-to-end PV calculation and validate the results

Finally, I ran Atlite’s full PV pipeline using ERA5-Land cutouts for Malaysia. I verified that the workflow runs without manual fixes and produces reasonable results. This step creates a dataset with the PV potentials that I can use to perform analysis. 


---

## Results


### 1. Does the new dataset improve data resolution?

<img src="/notebooks/atlite_era5_land_analysis/Data_resolution.png" width="1000px" height="300px" alt="Alt text for the image" />

*Note: You might need to open the image in another tab to see it clearly.*

The difference in spatial resolution is immediately visible. The ERA5 result (left) looks blocky, with clear grid-like boundaries between neighbouring cells. In contrast, ERA5-Land (right) produces a much smoother surface with finer spatial detail.

This achieves the primary objective behind the project: **to produce higher-resolution PV estimates for Malaysia**. With ERA5-Land’s smaller grid cells, the PV potential map becomes much more useful for real-world decision making. For example, it enables:

- better identification of local PV hotspots
    
- more realistic comparisons between nearby regions
    
- improved state-level and district-level analysis
    
- stronger inputs for site screening and feasibility studies

&nbsp;

### 2. Does this match existing analysis?
As a quick sanity check, the PV yields produced by Atlite fall within a realistic range. If we take the annual PV prediction (in **kWh/kWp/year**) and convert it into an average daily yield, we get:

- **1250 kWh/kWp/year → ~3.4 kWh/kWp/day**
    
- **1500 kWh/kWp/year → ~4.1 kWh/kWp/day**
    

These values are reasonable for Malaysia’s tropical climate, where solar resources are generally strong but moderated by frequent cloud cover and seasonal rainfall.

To validate this further, I cross-referenced the results against an independent source ( https://www.solarsunyield.com/latestnews/nid/167598/) summarising PV yield trends across Malaysia. The reported daily and yearly yields fall within the same range, and the regional patterns are consistent with what my results have shown, with northern and central regions generally performing better than the southern peninsula and East Malaysia.

This source also uses the **Global Solar Atlas**, which provides a global summary of PV potential. While the **Global Solar Atlas** uses its own modelling assumptions and input datasets, the overall magnitude and geographical trends align well with my output, giving additional confidence that the ERA5-Land integration with Atlite is producing sensible PV estimates.

![Data resolution comparison](/notebooks/atlite_era5_land_analysis/Solar_sun_yield_website.png)

My map:
![Average yearly yield](/notebooks/atlite_era5_land_analysis/Average_yearly_yield.png)


As you can see the values from my PV prediction aligns with the geographical trends and the daily trends from the referenced sources. 

&nbsp;

### 3. Which states in Malaysia are optimal for solar farms?

![PV by state](/notebooks/atlite_era5_land_analysis/PV_by_state.png)

This chart shows the **average annual PV yield** in each state, measured in **kWh/kWp** (which means _the amount of kWh of electricity you can generate in a year for every 1 kW of solar panel installed_). The kWp is the maximum theoretical output that can be produced under ideal test conditions (1000 W/m squared solar radiation, 25 degrees Celsius), which is very much different from actual real-life implementation. 

To make the results easier to interpret, I aggregated the PV estimates by state and compared the average yearly yield across each state.

Based on the bar chart above (sorted by yearly mean PV yield), the northern states such as **Perlis, Pulau Pinang, and Kedah** consistently rank among the highest. Central regions such as **Selangor, Putrajaya, Kuala Lumpur, and Melaka** also show strong PV potential.

On the other end of the spectrum, **Johor and Sarawak** have the lowest predicted PV yield, which may reflect regional differences in cloud cover and rainfall patterns.

Overall, most states fall within a relatively narrow band of roughly **1300 to 1500 kWh/kWp per year**. For context, this is generally higher than many parts of Europe (where values around ~900–1200 kWh/kWp/year are common), and is comparable to strong solar regions such as parts of Southern Europe, Australia, and the southwestern United States. This reinforces that Malaysia has broadly strong solar conditions.

---

## Future work

Here are some limitations of the analysis:
1. The results above show how much energy (in kWh) could be generated in a year if we assume **1 kW of installed PV capacity in every grid cell**. In other words, this is a _resource potential map_. It describes how good the solar conditions are, but it does not assume how much solar capacity is actually built today. Because of this, we cannot draw conclusions about the performance of Malaysia’s existing solar farms, or how closely current deployment aligns with the highest-potential regions.

2. Also, many areas are not feasible for large-scale PV deployment. Some locations may be constrained by land use and zoning rules, dense urban development, steep terrain, protected forests, wetlands, or simply lack of grid access. These constraints mean that high potential does not automatically translate into buildable capacity.


For future work, I plan to extend the analysis into these directions:

1. Firstly, I want to **introduce land-use and feasibility filtering** so that the results focus on areas where PV is realistically buildable. 
2. Secondly, I want to **incorporate datasets that describe where solar farms have already been built** so that the PV estimates can be linked to real deployment patterns. 
3. If possible, I also want to **obtain historical electricity generation data**, so that the Atlite results can be cross-validated against real-world output.

These additions would make the model more grounded and provide significantly more confidence in the PV estimates.

I may also open a PR to contribute this ERA5-Land integration back into Atlite. Before doing so, I want to spend more time aligning the implementation with Atlite’s conventions, improving test coverage, and documenting the dataset assumptions. Depending on the scope expected by the maintainers, I may also extend the module beyond PV to support additional features such as wind-related variables.

---

## Conclusion and Reflection

This project is a rewarding culmination of several months of work, and I’m glad to have achieved my main goal of producing a high-resolution PV estimate for Malaysia.

Further work will determine how well these estimates hold up against real-world deployment and PV generation data, but I hope this project can already provide useful insight into Malaysia’s solar energy potential.


If you would like to view the work, you can do so in the following links:
1. EDA script: [ERA5_Land_EDA](/notebooks/atlite_era5_land_analysis/era5_land_EDA.html)
2. Atlite fork: https://github.com/GohNgeeJuay/Atlite_ERA5_Land






