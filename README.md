# Introduction

In Los Angeles County alone, there are over 6,000,000 trees, making them a crucial component of the local ecosystem and infrastructure. Given the challenges posed by climate change, drought, and natural disasters, particularly for vulnerable populations, understanding and monitoring tree health is increasingly important. Trees play a significant role in mitigating these challenges by cooling neighborhoods, improving air and water quality, and enhancing community well-being. However, the rate of tree mortality surpasses the pace of replacement, necessitating effective monitoring strategies.

Remote sensing technology presents a promising solution for assessing urban forests' health and identifying tree mortality. By leveraging spatially explicit data from satellites, scientists and stakeholders can monitor large geographic areas cost-effectively and efficiently. While previous methods relied on manual field assessments, remote sensing allows for broader coverage and more frequent updates. Specifically, advances in spaceborne remote sensing have enabled the measurement of urban canopy structures and tree cover change, albeit with limitations in identifying individual tree health or mortality.

In this context, this study presents a workflow for creating an NDVI time series for any tree species, both for individual crowns and at the neighborhood scale, using high resolution Sentinel-2 satellite imagery. The Normalized Difference Vegetation Index (NDVI) is a widely used metric in remote sensing to assess vegetation health and vigor. It measures the difference between the near-infrared (NIR) and red bands of the electromagnetic spectrum, normalized by their sum. This index provides valuable insights into vegetation density, photosynthetic activity, and overall plant health. High NDVI values typically indicate dense, healthy vegetation, while low values suggest sparse or stressed vegetation. NDVI is particularly useful in monitoring changes in vegetation over time. To exemplify this process, NDVI time series were created for Quercus agrifolia (coast live oak) across Altadena, California.

# Data

This methodology employs Sentinel-2 satellite imagery. Launched by the European Space Agency (ESA) as part of the Copernicus program, Sentinel-2 captures high-resolution optical images of the Earth's surface with a revisit time of five days. Equipped with multispectral sensors, Sentinel-2 observes the planet in various spectral bands, enabling the monitoring of land cover, vegetation health, agricultural practices, and environmental changes. Its 10-meter spatial resolution facilitates detailed analysis at regional and even local scales. Moreover, its open-access policy ensures that researchers, policymakers, and the public alike can harness its data for diverse purposes, ranging from disaster response to urban planning.

LIDAR data was also utilized to delineate individual tree crowns. LIDAR, an acronym for Light Detection and Ranging, is used extensively in fine-scale geospatial mapping and environmental analysis. LIDAR works by emitting laser pulses (often from an aircraft)and measuring their reflection. LIDAR systems provide highly accurate and detailed three-dimensional representations of terrain, structures, and vegetation and LIDAR data is often used to map tree canopy in both urban and forest environments. This application uses LIDAR data from the Los Angeles Region Image Acquisition Consortium (LARIAC). The LARIAC dataset is proprietary aerial imagery commissioned by Los Angeles County in contract with the Pictometry International Corporation. Flights for the County have been flown every 2 to 4 years starting in 2002. The most recent flight was in 2020 using the EagleView-6 camera sensor suite. The full dataset covers fine-resolution multispectral (three- and four-band) imagery and discrete-return LiDAR point clouds.

# Methodology

## Google Earth Engine
1. Following delineation of street tree crowns and classification of tree species in Altadena, Quercus agrifolia tree crowns were imported into Google Earth Engine, as well as the Altadena neighborhood boundary. Google Earth Engine is a cloud-based platform for planetary-scale environmental data analysis. The colossal computing infrastructure of Google Earth Engine provides capabilities to run geospatial analysis at an unprecedented scale. It offers a vast array of public datasets that include satellite imagery, geospatial datasets, and client-side functions for manipulating and analyzing data. It performs on-the-fly computations, allowing users to visualize the results without first downloading and processing raw data locally. 
2. Sentinel-2 Level 2A Surface Reflectance imagery is imported into the same script, as well as Sentinel-2 Cloud Probability imagery. 
3. The surface reflectance imagery is then clipped to Altadena, and filtered to only include imagery from January 1, 2017 to March 31, 2023. 
4. Next, two cloud-masking functions are applied to remove imagery on cloudy days. The first uses the Q60 band from the surface reflectance imagery, while the second uses the cloud probability imagery to filter out days where cloud probability is greater than 10 percent. 
5. Another function is then applied to calculate NDVI across the cloud-masked imagery. 
6. After calculating NDVI, the imagery is clipped to only the Quercus agrifolia tree crowns. 
7. This imagery is then converted into a table, with each row representing a Quercus agrifolia crown on a specific date, and this table is exported as a csv file to Google Drive for use in the next script, which develops the interactive application.  Note: This table can also be exported directly to a Google Earth Engine asset, the purpose of exporting to Google Drive is to enable download onto a computer for use in the Python script as detailed in the next section. 
8. In this script, the table with the Quercus agrifolia crowns and corresponding NDVI values for each date is imported and then further filtered to include only dates with cloud probability below 5 percent. The Quercus agrifolia crowns are then added to the map. 
9. Next, a new table is created with the average NDVI across all Quercus agrifolia crowns for each date. This is done by first generating a list of each unique date in the table, then creating a function which calculates the mean NDVI for a specific date and returns it as a feature (with null geometry). This function is mapped across the list of all dates in the table previously generated, creating the new table. 
10. Code is then implemented that allows the user of the app to click on a crown and automatically generate an NDVI time series chart for that individual crown, as well as a time series chart showing the average NDVI values for each date across all crowns. This code defines a new point feature where the user clicked, and then extracts the Quercus agrifolia crowns within 10 meters of where the user clicked. This is necessary, as at a zoomed-out scale it is difficult for the user to click exactly within the bounds of a crown, and often crowns are clumped together or even overlap. A scatter chart is then generated which shows the NDVI for the extracted crown(s) on the y-axis and the date on the x-axis, as well as a legend showing species. 
11. Finally, another code block generates a chart showing the average NDVI for each date across all Quercus agrifolia crowns in Altadena, for comparison with the crown selected. The x-axis and y-axis bounds are also set to be the same within the code. 

## Python
1. If Python (version 3.11 or higher) was not already installed, it was downloaded and installed from Python.org.
2. If necessary, required packages were installed using the computer terminal with commands such as:

```{include=TRUE}
pip install numpy
pip install pandas
pip install matplotlib
```

3. Necessary packages, including numpy, pandas, matplotlib.pyplot, and matplotlib.dates, were imported.
4. Variables were created for the rolling frequency (for calculating rolling mean), z-score for the desired confidence interval for plots (1.96 was used for a 95% confidence interval), and NDVI decrease threshold to identify unhealthy trees (0.3 was used for Quercus agrifolia).
5. The CSV file with NDVI values for each date for each tree crown, as generated in Google Earth Engine, was imported.
6. Rows with NA values or NDVI < 0.1 were filtered out, and the date column was converted to Python format.
7. A table was created with the daily mean NDVI across all crowns, and rows with NA values or mean NDVI < 0.1 were filtered out.
8. Rows with daily mean NDVI < 0.1 were filtered out from the table with all tree crowns for each date.
9. All tree crowns with NDVI on the y-axis and date on the x-axis were plotted. The confidence interval and rolling mean were calculated and included in the chart, which was exported to the working directory.
10. The confidence interval and rolling mean were plotted without the points for individual tree crowns, and the plot was exported to the working directory.
11. The mean NDVI for each tree was calculated, and a figure with a boxplot showing NDVI values for each tree, with tree ID on the x-axis, was created. The x-axis was sorted from the smallest mean NDVI to the largest mean NDVI, and the plot was exported to the working directory.
12. Using a for loop, trees with NDVI drops above the specified threshold were identified. From the table with all tree crowns, only the identified trees were selected, and a new table was created from this selection. The new table was exported to the working directory. 
13. The rolling mean and confidence interval for the daily mean NDVI table were calculated, and the upper and lower bounds of the confidence interval were used to identify high and low outliers. A scatterplot of daily mean NDVI values, with the confidence interval, rolling mean, high outliers, and low outliers differentiated by point color, was created. The plot was exported to the working directory.


# Implementation

The interactive application runs on the computational power of Google Earth Engine, a cloud-based platform for planetary-scale environmental data analysis. Anyone with the link can view and interact with this application. 

# Known Issues

- If a user clicks on a tree that is not Quercus agrifolia, the species is printed on the side panel. However, an error will come up for the NDVI plots. Likewise, several error messages may be printed on the side panel if the user clicks on a point that is not a tree. 
- Each time the user clicks on the map, the Quercus agrifolia crowns will be added to the map again.


# Benefits and Applications

This GEE application offers numerous benefits, including:

- Enhanced Monitoring and Management: Facilitates the monitoring of urban tree health and biodiversity, enabling more informed management decisions.
- Educational Tool: Serves as an educational resource for schools and the public, raising awareness of the importance of urban forests.
- Research and Development: Provides a valuable dataset for academic research on urban ecology and urban planning.
- Policy and Planning: Assists policymakers in developing strategies for urban forest expansion, maintenance, and sustainability.




