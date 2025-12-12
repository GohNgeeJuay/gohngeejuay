---
title: "Deep Dive into solar energy modelling"
date: 2025-12-12
author: "Jake Goh"
summary: "An in-depth exploration into Top-of-Atmosphere irradiance in solar energy modelling"
showToc: true
tocOpen: false
draft: false
---

## Introduction: 


One of the reasons I’m fascinated by data is its ability to illuminate the natural world and help us understand complex, real-world phenomena. Renewable energy is one of those domains where data is not just interesting, it can help us understand abstract environment variables and gives us insight that can shape our decisions for the future. 

One area that particularly interests me is understanding how **climate data can be translated into actual, measurable energy potential**. While projects like **Atlite** already offer powerful tools for large-scale renewable energy modelling, I wanted to explore the fundamentals myself and build something from the ground up. Doing so gives me the chance to understand how climate modelling is done. 

This project is my attempt to delve into the topic of solar energy modelling. I aim to **understand, reconstruct, and validate** the core steps behind **solar energy modelling** using publicly available climate datasets. I would like to preface this in saying that I'm by no means an expert in climate or renewable energy modelling, and I am approaching this from a more data engineering/analytics perspective. 

---

## **Goals**


1. **Deep-dive into the foundations of solar energy modelling**  
    I want to learn the what are the different components for solar energy modelling, and how do they interact with each other to provide estimates. 

2. **Build a robust, transparent, and well-tested pipeline**  
    There’s a saying: _“It’s better to have no data than the wrong data.”_  
    With that in mind, I’ve invested significant effort into validation and unit testing to ensure that my results are trustable. I would say about 70% to 80% of my time was dedicated to testing and doing exploratory data analysis to make sure my data is sound. 
    
3. **Enable scalability across time, datasets, and regions**  
	The long-term vision is to support wider spatial regions, and longer time periods — with the possibility of extending the pipeline for forecasting and integration with Atlite. 
    

---



## **Why Start With Malaysia?**


Malaysia provides a compelling starting point for this work. In 2023, Malaysia introduced the **National Energy Transition Roadmap (NETR)**, which outlines the country’s pathway toward achieving **net-zero emissions by 2050**. Solar energy is envisioned to play a major role, given Malaysia's solar potential of about 269 GW, although only a small fraction of this is realized. Some of NETR goals are: 

- scaling up large-scale solar (LSS) installations,
    
- achieving over **40% renewable energy share in the national power mix by 2035**, and
    
- achieving 70% RE share of installed capacity by 2050, predominantly driven by solar PV installation. 
    


I did some research to find datasets that can provide insights into the following:

- which regions of Malaysia receive the most sunlight,
    
- and how these patterns might translate into solar PV potential.
    
This would allow us to understand and plan in which regions we should concentrate our efforts to harness solar energy. 

My goal is to gather the necessary data and build the methodology needed to begin answering these questions, starting from the first principles rather than relying on existing tools. By doing so, I would get a solid foundation in understanding solar energy modelling.



## Problem statement


For this project, I evaluated widely used climate datasets such as **SARAH (Surface Solar Radiation), ERA5**, and **ERA5-Land**. 

SARAH have a high spatial resolution (**~0.05°**, ~5 km), but mostly the coverage is focused on Europe and Africa. There is no coverage in this part of the world, which means this dataset is not usable. 

![SARAH](/notebooks/solar_toa_influx_analysis/SARAH.png)


ERA5 seems like a good choice since it has global coverage and it has many climate variables such as irradiation, temperature, wind, humidity, etc. The issue is it's resolution is  (~0.25°, ~31 km grid). This spatial resolution is too coarse for this project. 

ERA5-Land is based on ERA5 but it is running on an enhanced spatial resolution (~0.1°, ≈ 9 km). Overall, this would be a good pick because it has the required coverage in Malaysia and the variables needed. It is also compatible with Atlite pipeline to estimate solar energy potential, and this dataset was picked as the main dataset. Below, you can see a comparison of a selected variable and the granularity.  


![ERA5_Comparisons](/notebooks/solar_toa_influx_analysis/ERA5_Comparisons.png)



## Solution


For some background, solar radiation is the energy coming from the sun into the space in the form of electromagnetic waves, mainly visible light and shortwave radiation. This energy is the source of electricity in the solar PV systems. There are 2 categories of solar radiation that we are concerned with: 


**Top of Atmosphere.** 
This is the amount of radiation arriving just outside the atmosphere. It is affected by the geometry of the earth and its relation to the sun. Factors affecting it is only geometrical:
1. Latitude
2. Date and time during the year
3. Solar position in relation to the earth
4. Earth-sun distance.


**At the surface**
This is the amount of radiation that actually reaches the ground after passing through cloud, water vapor, dust, atmospheric gases, etc. This is what is received by the solar panels, and it can vary a lot based on the climate and weather. 


ERA5_Land provides variables describing the radiation at the surface such as Surface net short-wave (solar) radiation (SSRD) which describes the amount of solar radiation that reaches a horizontal plane on the surface of the Earth after reflected passing through atmosphere. 

Since I wanted to delve deeper into solar energy modelling, I calculated the TOA-influx from scratch. TOA influx is the physical upper bound for all solar radiation values since it is radiation before passing through the atmosphere. Because TOA irradiance is determined purely by solar geometry and physics, it served as an independent benchmark to compare against ERA5-Land’s modeled surface radiation measurements which applied many other variables on the incoming TOA radiation. 


**How to calculate TOA Influx.** 

Here is the formula to calculate TOA Influx. Note that the formula is strictly geometrical, and no other variables are included in the formula. 
![Formula_1](/notebooks/solar_toa_influx_analysis/Formula_1.png)
![Formula_2](/notebooks/solar_toa_influx_analysis/Formula_2.png)
![Formula_3](/notebooks/solar_toa_influx_analysis/Formula_3.png)

Since the aim of my project is a more practical goal rather than a deep dive into solar geometry research, I did not delve too much into the details. 

My goal was just to get a consistent TOA measurement, and ensure that it "make sense" by cross validating with existing datasets like ERA5 dataset. The calculated TOA Influx should also be in the same format expected by tools like Atlite, and this calculation should be reproducible across the dataset. 

I mainly approach this with a data engineering mindset rather than focusing on the solar physics. So I relied on unit testing, sanity checks and visual comparisons rather than any complex validations. This allowed me to produce a TOA dataset that is **scientifically sound, practical, and and reproducible, without over-engineering a component that is meant to be a baseline reference. 


Also, to help narrow the focus, I focused on implementing the formula onto the year of 2022, and focus on Peninsular Malaysia. This is not the full extend to the project, but I did it in such a way so I can focus on a smaller geographical region and time period so that testing can be less daunting and I can extend it further once I am satisfied with the results. 


## Design 


![Design_flowchart](/notebooks/solar_toa_influx_analysis/Design_flowchart.png)


### **1. Download Climate Datasets**



The workflow begins with obtaining the required climate reanalysis data from **ERA5-Land** and ERA5 using the Copernicus API via CDS. I will be adding the new calculation on top of ERA5-Land, and using the ERA5 as to cross validate since it also have a variable that measures the top-of-atmosphere radiation. 


How I did this was using the CDSAPI library, which is a python library that allows users to programmatically access the Copernicus Climate Data Store. Below is an example code that request for multiple months of data from the API, and specifying what data variables I need, data format, area of interest and other variables. 


![API_download](/notebooks/solar_toa_influx_analysis/API_download.png)



### **2. Preprocessing**


The next step was to perform preprocessing step so that I will get a clean and analysis-ready dataset that I can use. Since Atlite uses Xarray as the library of choice for the processing of data, my goal was to get the downloaded data into a format that can be used by Xarray. 

Xarray is kind of like the more popular Pandas, but I find it significantly more intuitive to use. It is particularly useful for multi-dimensional data such as for physics, astronomy, bioinformatics and geoscience. 

Some of the steps taken is to combine the monthly files into a single data file, zipping, renaming and other standard data processing steps. An example is below. Note than the resulting variable is a valid Xarray Dataset. 

![Preprocessing](/notebooks/solar_toa_influx_analysis/Preprocessing.png)



### **3. TOA Influx Calculation**


ERA5-Land does not provide Top-of-Atmosphere (TOA) irradiance, so this value must be calculated manually. I have implemented the formula as mentioned in the previous section. Each function is separated which allows for easier testing and debugging. 


An example is below. I have separated the complex formula into simple functions such as to calculate solar zenith angle, hour angle, etc. Note that at the last function (calc_toa_incident_shortwave), there is a new variable representing TOA influx calculated. This new variable is calculated for every grid cell and timestamp. 
![TOA_calculation](/notebooks/solar_toa_influx_analysis/TOA_calculation.png)



### **4. Unit Testing**


Because this project involves custom scientific calculations, unit tests are essential for maintaining correctness and reproducibility. For example, tests include:

- Verifying nighttime TOA values drop to zero
    
- Ensuring solar zenith angle (the angle between the sun's position and the vertical directly overhead) is 0 degrees when directly overhead and 90 degrees when it is at the horizon. 

![SZA](/notebooks/solar_toa_influx_analysis/SZA.png)


I have implemented the test using Pytest. Each function that I created in the previous section is tested against expected values based on physics and solar geometry. Below is an example: 

![Pytest_code](/notebooks/solar_toa_influx_analysis/Pytest_code.png)

This is how I run all of my tests:
![Pytest_run](/notebooks/solar_toa_influx_analysis/Pytest_run.png)



### **5. Validation and Sanity Checks**


Beyond verifying that each function behaves correctly, I also validate the results from a broader perspective. This involves formulating simple questions about how the data _should_ behave. I employ simple data visualizations such as time series plots or spatial maps to confirm that the outputs are logical. 

Whenever a chart reveals something unexpected, I trace the issue back to the relevant function, make corrections, and rerun the checks. This iterative process took time, but it ensured that the final dataset behaves consistently and aligns with known solar patterns.

To keep this section focused and readable, I present only the _corrected_ charts and interpretations rather than documenting every bug or error. 

---


Here are some questions:
1. **Surface level check: I have calculated the TOA irradiance values for each grid cell, but do they seem correct?**


After computing TOA irradiance for every grid cell, the first check was a simple visual sanity check: _Do the values follow expected solar behaviour?_  
For example, irradiance should peak around local noon, drop to zero at night, and vary smoothly across neighbouring grid cells. Any abrupt spikes or physically impossible values (e.g., negative irradiance) would immediately indicate an error.


For this, I selected a particular date and plotted a choropleth for a few hours in the day. The resulting image below shows the TOA influx/irradiance values per grid cell at the time 8am, 12pm, 4pm and 8pm local time in the first of January 2022. This seems good as the trend is what I was expecting. The TOA influx grows in the morning, peaks around noon and falls in the evening. The values also vary smoothly across neighboring cells, and we do not see any weird banding or missing values.  

Note that the values for the different hours are using different scales, which does not allow us to make direct comparisons of values across time. We shall do this in the next question. 
![Spatial_validation](/notebooks/solar_toa_influx_analysis/Spatial_validation.png)



---

2. **Does a selected grid cell match ERA5 when compared directly?**  
ERA5-Land does not natively provide TOA irradiance, so I used ERA5 (which _does_ include TOA) as the benchmark.  
For any specific latitude–longitude point, the calculated ERA5-Land-based TOA values should align closely with the ERA5 equivalent, with no major structural differences. We are expecting the irradiance to grow in the morning, peak at noon and fall in the evening. The trends and daily cycles should overlap between the two datasets.


What I did here was to pick a random latitude and longitude (3.13 and 101.58), and plotted the hourly TOA influx from both datasets. 

![Temporal_validation](/notebooks/solar_toa_influx_analysis/Temporal_validation.png)

This looks good! The daily trend seem to match and we are seeing the bell shape at every day. 

But we can see that for the same date and hour, our calculated values are higher than the ERA5 TISR. It seems like our calculations are "predicting ahead". 

This happens because of how ERA5 defines accumulated fields like **TISR**.  
**TISR is an accumulated value over the hour _ending_ at the timestamp**.  
So, for example:

- **ERA5 TISR at 2022-01-01 08:00** represents the _total TOA radiation from 07:00 → 08:00_.
    
- **Our calculated TOA influx at 08:00** represents the _incoming radiation for the next hour, 08:00 → 09:00_. 


*Note: It is important to note that our method assumes the sun–Earth geometry (e.g., solar declination, and solar zenith angle) is fixed over that one-hour interval. In reality, these parameters change continuously throughout the day. However, the variation within a single hour is extremely small, so this approximation introduces negligible error—and as shown above, our calculated values still match the ERA5 baseline very closely.*


Since the two values refer to _different 1-hour periods_, our series appears shifted forward relative to ERA5.

Another question might arise: 

 Why are the accumulated TOA influx values roughly the same even though ERA5 and ERA5-Land have very different grid sizes?

This is because both datasets report TOA shortwave radiation in **J/m²**.  
This unit represents _energy per unit area_. The physical grid-cell size does not affect the value, because the energy is already normalized per square metre.

So even though ERA5 grid cells are much larger than ERA5-Land cells, the accumulated irradiance per m² should be the same (aside from small interpolation differences), since TOA radiation is a function of solar geometry, not spatial resolution.



---

3. **Do monthly totals agree between ERA5 and ERA5-Land?**  
While hourly values may differ slightly, _aggregate behaviour_ should remain consistent.  

To check whether both datasets behave consistently at a broader scale, I aggregated the hourly TOA irradiance into **monthly totals** and then computed the **average monthly total** for each grid cell. This gives a spatial map of how much TOA energy a location typically receives over a month.

![Monthly_totals](/notebooks/solar_toa_influx_analysis/Monthly_totals.png)

The plots show that the values for both datasets are very close. They have similar spatial patterns with the southern region of Peninsular Malaysia having slightly higher irradiance than the north. 

Although the ERA5 TISR values are slightly higher overall, this difference is very small and consistent throughout the region. Keeping in mind that this is the monthly accumulated totals, even small hourly mismatch can compound. Therefore, this small positive bias is not a source of concern due to the methodological difference between our simpler TOA model with ERA5 model. 



---
4. To quantify agreement, I computed standard error metrics such as RMSE, MAE, MBE, NRMSE, NMAE, and correlation (R).  
In general:

- **Root Mean Square Error (RMSE) / Mean Absolute Error (MAE)**: This measures the absolute errors between predicted values and actual values. The lower the better; calculated values should be small relative to the mean irradiance.
    
- **Mean Bias Error (MBE)**: This measures the average difference between forecasted values with actual values. This should be close to zero (indicating no systematic over- or underestimation).
    
- **Normalized Root Mean Square Error (NRMSE) / Normalized Mean Absolute Error (NMAE)**: This values is the same as RMSE and MAE but it is standardized by the range of the data. This would make it less scale-independent and comparable across different units and magnitudes. It should ideally be below 10–20% for climate-physics variables.
    
- **Correlation (R)**: should be very high (typically >0.95) because TOA radiation is deterministic and geometry-driven.
    

These metrics help confirm that the calculated values are not only visually plausible but also numerically consistent with an established reference dataset.



#### ***How the metrics were computed***

Because ERA5 and ERA5-Land have different spatial resolutions, I first **reindexed the ERA5-Land dataset onto the ERA5 grid**. This ensures that:

- every ERA5-Land grid cell is aligned with the nearest ERA5 cell,
    
- both datasets share the same latitude/longitude coordinates,
    
- point-to-point comparisons are meaningful.
    

After aligning the grids, each metric was computed by comparing:

> **Each grid cell in ERA5-Land vs. the corresponding cell in ERA5, across all hourly timesteps.**

This means the statistics reflect **spatio-temporal error**, not just a single location or a single time slice. It’s effectively a full-dataset consistency check.

Here is how the grid sizes look like for both datasets after reindexing. 
![Reindexing_ERA5](/notebooks/solar_toa_influx_analysis/Reindexing_ERA5.png)

![Reindexing_ERA5_Land](/notebooks/solar_toa_influx_analysis/Reindexing_ERA5_Land.png)

Now I calculated the values for each cell against the same cell and across time. Here are the results:

| Metric | Value |
| :-------- | ----: |
| RMSE | 302553.81 |
| MAE | 195104.71 |
| MBE | 3144.27 |
| NRMSE | 0.0615 |
| NMAE | 0.0396 |
| R | 0.986 |


#### ***Interpretation***

- **The RMSE and MAE values are small relative to the typical daily TOA peak (~4–5×10⁶ J/m²).**  
    This indicates that absolute deviations remain modest.
    
- **MBE is close to zero (relative to the magnitude of the data)**. The peak TOA value is 4 to 5  million J/m² while this is around 3144 J/m² which is less than 1%. This means  the method does not introduce any meaningful systematic bias—no consistent “overprediction” or “underprediction.”
    
- **NRMSE (~6%) and NMAE (~4%)** are well within the acceptable range for climate-physics variables, especially when comparing datasets with slightly different handling of solar accumulation windows.
    
- **Correlation (R ≈ 0.986)** is extremely high, confirming that the temporal patterns—sunrise timing, peak irradiance, seasonal behaviour—are captured almost perfectly.  
    This is expected, since TOA geometry is deterministic, but it provides strong validation that the implementation is correct.


Overall, the statistical results strongly support that the implementation of TOA influx is both **physically consistent** as validated by the above charts and **numerically robust**.


---



### **6. Atlite Integration**


Once the dataset is validated, the next step is to integrate it into **Atlite**, a Python-based framework widely used for renewable energy modelling. So far, this project computes the **top-of-atmosphere (TOA) irradiance** which is the incoming solar energy before it interacts with the atmosphere. However, as discussed earlier, the solar energy that actually reaches the ground is significantly reduced and modified by atmospheric effects such as clouds, aerosols, water vapour, and scattering.

By integrating this dataset with Atlite, the pipeline can incorporate these additional atmospheric and surface variables (from ERA5-Land or other sources), allowing Atlite to compute **surface-level solar potential** rather than TOA-only values. This bridges the gap between theoretical extraterrestrial irradiance and the realistic solar energy available for photovoltaic modelling.


What it might involve:

- Mapping ERA5-Land variables to Atlite’s expected variable names, dimensions, and format. 
    
- Registering the custom dataset with Atlite’s data interface so that it behaves like other existing datasets like ERA5, SARAH, etc that are supported by the library 
    
- Connecting the current functions into Atlite's existing workflow 
    
- Verifying the workflow produces correct results across different locations and time ranges. 
    


### **7. Surface Solar Energy Output**


The output of the Atlite will consist of **spatially explicit, hourly PV potential values**, which can support downstream use cases such as:

- PV site selection - to identify high-yield solar farm development
    
- Renewable-energy planning - supporting long-term resource assessment and policy planning.
    
- Energy system modelling   - integrating realistic solar energy estimation into power system simulation. 

This workflow will help to bridge the gap between climate datasets and practical insights for the renewable energy sector.  


# Resources

If you would like to view the jupyter notebook output, you can view them here:
1. Download script: [TOA_influx](/notebooks/solar_toa_influx_analysis/download_data.html)
2. EDA script: [TOA_influx](/notebooks/solar_toa_influx_analysis/toa_influx.html)

To view the full code including the data, notebooks, functions and tests, click here: https://github.com/GohNgeeJuay/TOA_Influx_Analysis. 



# Conclusion and Reflection


This project has gone through several rounds of validation and experimentation and I have reached a milestone where I feel confident enough to share the work more broadly. While I am committed to produce quality result, I am also balancing my thoroughness with iterative progress. 


There is still plenty of room to improve, and optimize. I thank you for reaching the end, and I welcome any feedback or questions that you have. 

### Contact info
You can reach me via GitHub: https://github.com/gohngeejuay

Or linkedin: www.linkedin.com/in/gohngeejuay

This repository is licensed under the [CC BY 4.0](LICENSE) license.



