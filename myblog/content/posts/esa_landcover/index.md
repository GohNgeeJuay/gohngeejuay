---
title: "Land cover data"
date: 2026-08-31
author: "Jake Goh"
summary: "Augmenting land-suitability analysis using ESA WorldCover data"
showToc: true
tocOpen: false
draft: false
---

## 1.  Problem statement

As part of deciding where to build solar capacity, we need to look at prospective locations. Some places already contain built-up such as buildings, where solar capacity may be more applicable to only rooftop solar. 

If we are interested in large scale solar capacity building such as solar farms, we might focus on places where there is high potential PV and also places where land cover is more compatible such as places where there is sparse vegetation, less tree cover or bare ground compared to places with dense tree cover or water bodies. These factors do not mean that the location can be developed with solar capacity, but it provide useful context when comparing different sites. 

To do this, I added another data layer to my dashboard. I used the [ESA WorldCover data](https://esa-worldcover.org/en/data-access) to provide landcover data. This data will help to show what the surface consists of, such as trees, built-up/urban areas, cropland, wetlands, etc. 


## 2. Approach

To decide on how to use the landcover data, I focused on a few aspects:

### a.  Data granularity

I thought of a few ideas of the presentation of the data. Since the goal was to show the landcover, I initially thought of displaying the raster data as it is, though I realized that the granularity is way too detailed. The landcover data is on a 10m pixel, while the PV potential is based on a much coarser grid, with cells approximately 9km by 9km (based on ERA5 Land). Displaying the ESA data as it is would be way too noisy, and whether Streamlit can even support such rendering is another issue.


As you can see below, the data is quite granular even when zoomed out. The image below is showing the data for the Klang Valley area.  
![ESA_granularity](/notebooks/esa_landcover/ESA_granularity.png)

As a comparison to my current resolution, the same geographical area does not come close to the same level of detail. 
![ERA5_granularity](/notebooks/esa_landcover/ERA5_granularity.png)

I decided to aggregate the land-cover data to the resolution of ERA5 Land used by my PV model. For each cell (latitude and longitude), it will contain the landcover categories as a percentage of the total area within the cell. Thus the granularity of the data will be on a ~9km resolution. I will lose some granularity from aggregating, though I believe that is the appropriate and sufficient resolution for my current use case.  This would add another data layer that would be meaningful without putting a strain on the dashboard rendering and memory usage. 


### b. Data storage

Throughout the project, my decision was to store all data needed for the dashboard in parquet files, already pre-aggregated and ready for consumption. The reason was that I wanted the dashboard to quickly load any necessary data to display, and to move the computation intensive part to the ETL process. This would keep the user experience smooth.

Also, since I'm deploying the dashboard on Streamlit Community Cloud, I need to be conscious of the memory usage. It has a limit of [2.7GB of memory](https://docs.streamlit.io/deploy/streamlit-community-cloud/manage-your-app#resource-limits), so I need to ensure the data that is loaded and processed on the application side to be space-conscious as possible. 

For my existing data, I have a Parquet file containing the PV potential, and another 2 datasets containing the districts' and states' geographical information. I chose to store the data as parquet files since they are small in size and widely supported, including by Pandas which I used on the application side to load them as dataframes. This will push as much computation to the ETL process so that the Streamlit application can just focus on filtering, rendering and interacting with the preprocessed data. 

The goal of this ETL process should also be to create another Parquet file with land-cover data, ideally only containing the data required for the dashboard. 

### c. Data ingestion

To do this, firstly, I obtained the data from the ESA website. ESA WorldCover provides land-cover data at a 10m spatial resolution. The downloadable products are distributed as 3° × 3° Cloud-Optimized GeoTIFF tiles in EPSG:4326.

I initially obtained the data manually through the WorldCover viewer, where you can manually select the cells of data through a UI as seen below.

![WorldCover_viewer](/notebooks/esa_landcover/WorldCover_viewer.png)


In hindsight, this was not most efficient approach, as I had to manually select multiple tiles to cover my area of interest. ESA also provides programmatic access through the `terracatalogueclient` package and S3 downloads, apart from other methods. This could be useful if I were to make the data acquisition process more extensible in the future. But since I am just using the data as a proof-of-concept and an initial attempt, I decided to keep the ingestion as straightforward as possible for now. 


### d. Integration with existing data

As mentioned above, the resolution also have to be accounted for. I created a script to convert process the data from the raw form downloaded from the website into a parquet file with columns of latitude, longitude and landcover percentages.

Each cell downloaded from the ESA WorldCover is in its own file, which is around 25 to 35 MB. Initially, I thought of combining all of the raster files into a single large raster file using python. However, that was impractical on my device due to the amount of data involved and memory required to process it. 


The main logic goes as follows:
1. For each cell in my existing PV data, I will check which ESA file contain the landcover data for that cell. This means that the PV cell must be contained in that file. 
2. For each PV cell, I use the center coordinates of the cell and created a spatial window around it.
3. Then, I and read the landcover data that falls within the window rather than loading the entire raster. 
4. Finally, I calculated the landcover percentage for each landcover category that was read from the window.

This will give me the landcover percentage for each cell.  The data will know look something like below. For each unique latitude and longitude of the PV grid, there is a dictionary of landcover category and it's corresponding percentage cover in that cell. 

![dataset_view](/notebooks/esa_landcover/dataset_view.png)


## 3. Sanity Check

After that, I performed  some sanity checks. I hand-picked several locations which I have understanding of the dominant land cover, such as urban areas, or crop land, forest areas. I then displayed the raster data for it, and compared it with the resulting landcover percentage to validate my findings. This is a good visual comparison of the data. 
![sanity_check](/notebooks/esa_landcover/sanity_check_percentage.png)

With the sanity checks successful, I proceeded with writing it to a parquet file. 


---

## 4. Solution

There were a few ideas I could have displayed the data. For now, I decided on displaying the information in a simple bar chart. When a user selects a cell, the dashboard will display a bar chart showing the percentage of each land-cover category within that cell. 

Below is a recording of how it looks like. 

<video style="width: 100%; height: auto;" controls>
    <source src="/notebooks/esa_landcover/recording_landcover_demo.mp4" type="video/mp4">
</video>


## 5.  Summary

In this update, I added land-cover information to provide more context when evaluating the areas with solar PV potential. I aggregated the original 10m ESA WorldCover data to the ~9km ERA5 Land resolution used by the PV model. Now each PV cell contain data on the percentage of its area covered by different land-cover categories. 

In the future, I will focus on making the ETL process more extensible and automated, and also looking to improve the UI elements and user experience. 

