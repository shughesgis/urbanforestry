# Introduction

In Los Angeles County alone, there are over 6,000,000 trees, making them a crucial component of the local ecosystem and infrastructure. Given the challenges posed by climate change, drought, and natural disasters, particularly for vulnerable populations, understanding and monitoring tree health is increasingly important. Trees play a significant role in mitigating these challenges by cooling neighborhoods, improving air and water quality, and enhancing community well-being. However, the rate of tree mortality surpasses the pace of replacement, necessitating effective monitoring strategies.

Remote sensing technology presents a promising solution for assessing urban forests' health and identifying tree mortality. By leveraging spatially explicit data from satellites, scientists and stakeholders can monitor large geographic areas cost-effectively and efficiently. While previous methods relied on manual field assessments, remote sensing allows for broader coverage and more frequent updates. Specifically, advances in spaceborne remote sensing have enabled the measurement of urban canopy structures and tree cover change, albeit with limitations in identifying individual tree health or mortality.

In this context, this study presents a workflow for creating an NDVI time series for any tree species, both for individual crowns and at the neighborhood scale, using high resolution Sentinel-2 satellite imagery. The Normalized Difference Vegetation Index (NDVI) is a widely used metric in remote sensing to assess vegetation health and vigor. It measures the difference between the near-infrared (NIR) and red bands of the electromagnetic spectrum, normalized by their sum. This index provides valuable insights into vegetation density, photosynthetic activity, and overall plant health. High NDVI values typically indicate dense, healthy vegetation, while low values suggest sparse or stressed vegetation. NDVI is particularly useful in monitoring changes in vegetation over time. To exemplify this process, NDVI time series were created for Quercus agrifolia (coast live oak) across Altadena, California.

# Data

This methodology employs Sentinel-2 satellite imagery. Launched by the European Space Agency (ESA) as part of the Copernicus program, Sentinel-2 captures high-resolution optical images of the Earth's surface with a revisit time of five days. Equipped with multispectral sensors, Sentinel-2 observes the planet in various spectral bands, enabling the monitoring of land cover, vegetation health, agricultural practices, and environmental changes. Its 10-meter spatial resolution facilitates detailed analysis at regional and even local scales. Moreover, its open-access policy ensures that researchers, policymakers, and the public alike can harness its data for diverse purposes, ranging from disaster response to urban planning.

![Sentinel-2 true color composite with Altadena, California outlined in yellow](/truecolor.png)

LIDAR data was also utilized to delineate individual tree crowns. LIDAR, an acronym for Light Detection and Ranging, is used extensively in fine-scale geospatial mapping and environmental analysis. LIDAR works by emitting laser pulses (often from an aircraft)and measuring their reflection. LIDAR systems provide highly accurate and detailed three-dimensional representations of terrain, structures, and vegetation and LIDAR data is often used to map tree canopy in both urban and forest environments. This application uses LIDAR data from the Los Angeles Region Image Acquisition Consortium (LARIAC). The LARIAC dataset is proprietary aerial imagery commissioned by Los Angeles County in contract with the Pictometry International Corporation. Flights for the County have been flown every 2 to 4 years starting in 2002. The most recent flight was in 2020 using the EagleView-6 camera sensor suite. The full dataset covers fine-resolution multispectral (three- and four-band) imagery and discrete-return LiDAR point clouds.

# Methodology

## Google Earth Engine
1.	Following delineation of street tree crowns and classification of tree species in Altadena, Quercus agrifolia tree crowns were imported into Google Earth Engine, as well as the Altadena neighborhood boundary. Google Earth Engine is a cloud-based platform for planetary-scale environmental data analysis. The colossal computing infrastructure of Google Earth Engine provides capabilities to run geospatial analysis at an unprecedented scale. It offers a vast array of public datasets that include satellite imagery, geospatial datasets, and client-side functions for manipulating and analyzing data. It performs on-the-fly computations, allowing users to visualize the results without first downloading and processing raw data locally. 

```var bounds = ee.FeatureCollection('projects/ee-sarahmhughes2000/assets/Altadena');
var crowns = ee.FeatureCollection('projects/ee-sarahmhughes2000/assets/trainAltadena-Quercus_agrifolia');
```

2.	Sentinel-2 Level 2A Surface Reflectance imagery is imported into the same script, as well as Sentinel-2 Cloud Probability imagery. 
```var s2sr = ee.ImageCollection(‘COPERNICUS/S2_SR_HARMONIZED’);
var s2Clouds = ee.ImageCollection(‘COPERNICUS/S2_CLOUD_PROBABILITY’)
```

3.	The surface reflectance imagery is then clipped to Altadena, and filtered to only include imagery from January 1, 2017 to March 31, 2023. 

```// Import Level 2A Surface Reflectance Sentinel-2 Imagery
var s2 = s2Sr.filterDate('2017-01-01', '2023-03-31') // Filter by date
.filterBounds(bounds); // Filter by study area boundary (Altadena)
```

4.	Next, two cloud-masking functions are applied to remove imagery on cloudy days. The first uses the Q60 band from the surface reflectance imagery, while the second uses the cloud probability imagery to filter out days where cloud probability is greater than 10 percent. 

```// Cloud mask function using the QA60 band to mask clouds
var maskClouds = function(image) {
  var QA60 = image.select(['QA60']);
  return image.updateMask(QA60.lt(1));
};

var createTimeBand = function(image) {
 return image.addBands(image.metadata('system:time_start'))
};

var MAX_CLOUD_PROBABILITY = 10;

function maskClouds2(img) {
  var clouds = ee.Image(img.get('cloud_mask')).select('probability');
  var isNotCloud = clouds.lt(MAX_CLOUD_PROBABILITY);
  return img.updateMask(isNotCloud);
}

// The masks for the 10m bands sometimes do not exclude bad data at
// scene edges, so we apply masks from the 20m and 60m bands as well.
function maskEdges(s2_img) {
  return s2_img.updateMask(
      s2_img.select('B8A').mask().updateMask(s2_img.select('B9').mask()));
}

// Map the 'maskCloud' function over the 's2' ImageCollection, as well as the edges function
var s2masked = s2
    .map(maskClouds)
    .map(createTimeBand); // Use '.map()' to run GEE function

//join cloud mask (based on cloud probability) with s2Clouds
var s2SrWithCloudMask = ee.Join.saveFirst('cloud_mask').apply({
  primary: s2masked,
  secondary: s2Clouds,
  condition:
      ee.Filter.equals({leftField: 'system:index', rightField: 'system:index'})
});

//apply second cloud mask
var s2noClouds = ee.ImageCollection(s2SrWithCloudMask).map(maskClouds2);
```

5.	Another function is then applied to calculate NDVI across the cloud-masked imagery. 

```// Use GEE NDVI function to map an 'NDVI' band over the 's2noClouds' ImageCollection 
var ndvi = function(image) {
  return image.addBands(image.normalizedDifference(['B8', 'B4']) // Select NIR and R bands
  .rename('NDVI')); // Rename the band from default 'nd' to 'NDVI'
};

// Map NDVI function over cloud masked s2
var s2CloudsND = s2noClouds.map(ndvi); 
```
[Composite image of mean NDVI across the study area, with Altadena outlined in black](/ndvi.png)

6.	After calculating NDVI, the imagery is clipped to only the Quercus agrifolia tree crowns. 

```// Clip NDVI to tree crowns
var clipCrowns = s2CloudsND
.map(function(image) {
  return image.clip(crowns); 
});
```

7.	This imagery is then converted into a table, with each row representing a Quercus agrifolia crown on a specific date, and this table is exported as a csv file to Google Drive for use in the next script, which develops the interactive application.  Note: This table can also be exported directly to a Google Earth Engine asset, the purpose of exporting to Google Drive is to enable download onto a computer for use in the Python script as detailed in the next section. 

```// Convert ImageCollection (imagery) to FeatureCollection (table)
var reduce = clipCrowns.map(function(image) {
  return image
    .reduceRegions({
      collection: crowns, // Reducing based on smaller area
      reducer: ee.Reducer.mean(),
      scale: 0.6, // Keep scale at NAIP resolution
    }).map(function(f) {
      var date = ee.Date(image.get('system:time_start'))
      var formattedDate = date.format('yyyy/MM/dd');
      return f.set('date', formattedDate);
  })
});

//Export to CSV
Export.table.toDrive({
    collection: reduce.flatten(), // Flatten collection of years
    description: 'S2_NDVI-Quercus_agrifolia-crowns', 
    fileNamePrefix: 'S2_NDVI-Quercus_agrifolia-crowns', 
    fileFormat: 'CSV'
});
```

8.	In a new script, the table with the Quercus agrifolia crowns and corresponding NDVI values for each date is imported and then further filtered to include only dates with cloud probability below 5 percent. The Quercus agrifolia crowns are then added to the map. 

```var oakCrowns = ee.FeatureCollection(“projects/altadena-trees/assets/S2_NDVI-Quercus_agrifolia-masked2”);
oakCrowns = oakCrowns.filter('MSK_CLDPRB <= 5');
Map.addLayer(oakCrowns, {color: 'white'}, 'Oak Crowns');
```
[Quercus agrifolia crowns within Altadena are shown in white, with Altadena outlined in yellow](/crowns.png)

9.	Next, a new table is created with the average NDVI across all Quercus agrifolia crowns for each date. This is done by first generating a list of each unique date in the table, then creating a function which calculates the mean NDVI for a specific date and returns it as a feature (with null geometry). This function is mapped across the list of all dates in the table previously generated, creating the new table. 

```// Get distinct dates in the feature collection
var dates = oakCrowns.distinct('system:time_start').aggregate_array('system:time_start');
// Function to calculate mean NDVI for a given date
var calculateMeanNDVI = function(date) {
  date = ee.Date(date);
  var filtered = oakCrowns.filterDate(date, date.advance(1, 'day'));
  var meanNDVI = filtered.reduceColumns(ee.Reducer.mean(), ['NDVI']).get('mean');
  return ee.Feature(null, {
    'Date': date 
    'MeanNDVI': meanNDVI
  });
};

// Map over the dates and calculate mean NDVI for each date
var avgOak = ee.FeatureCollection(dates.map(calculateMeanNDVI));
```

10.	Code is then implemented that allows the user of the app to click on a crown and automatically generate an NDVI time series chart for that individual crown. This code defines a new point feature where the user clicked, and then extracts the Quercus agrifolia crowns within 10 meters of where the user clicked. This is necessary, as at a zoomed-out scale it is difficult for the user to click exactly within the bounds of a crown, and often crowns are clumped together or even overlap. A new feature collection, ‘trees’, is created, composed of the crown(s) overlaying where the user clicked, and then merged with the ‘avgOak’ feature collection representing daily mean NDVI. A scatter chart is then generated which shows the NDVI for the extracted crown(s) as points and the daily mean NDVI across all crowns as a line, plotted on the same y-axis, and the date on the x-axis. 

```// Register a callback on the default map to be invoked when the map is clicked.
map.onClick(function(coords) {
// Add a red dot showing the point clicked on.
    var point = ee.Geometry.Point(coords.lon, coords.lat);
    var dot = ui.Map.Layer(point, {
        color: 'red'
    });
    map.layers().set(1, dot);
    
    // Sample tree at clicked point
    function getTrees(point) {
        var pointBuffer = point.buffer(10);
        var treesNearPoint = oakCrowns.filterBounds(pointBuffer);
        return treesNearPoint
      }
      
// Create new feature collection from crown(s) where user clicked
    var trees = getTrees(point)

	// Function to add a 'Date' property based on 'system:time_start'
var addDateProperty = function(feature) {
  	var date = ee.Date(feature.get('system:time_start'));
  	return feature.set('Date', date);
};

      	// Map over the individualNDVI collection to add the 'Date' property
        	trees = trees.map(addDateProperty);

	// merge datasets for plotting
        	var combinedData = trees.merge(avgOak).sort('Date');

 // Create a chart with points for individual tree NDVI values and a line for daily mean NDVI
var mergedChart = ui.Chart.feature.byFeature(combinedData, 'Date', ['NDVI', 'MeanNDVI']).setChartType('LineChart');

	 // Define start and end dates for the horizontal axis
    	 var startDate = ee.Date('2019-01-01').millis().getInfo();
   	 var endDate = ee.Date('2023-03-31').millis().getInfo();

// Customize the chart
mergedChart.setOptions({
  title: 'Daily NDVI for Selected Oak Crown and Daily Mean NDVI for all Oaks',
            	// Set the title of the chart.
           	 vAxes: {
               	 0: {
                    // Set the title of the vertical axis.
                    title: 'NDVI',
                    // Set the format of the numbers on the axis.
                    format: '#.##',
                    // set axis boundaries
                     viewWindow: {
                       min: 0,
                       max: 1
                      },
                    // Set the style of the text.
                    titleTextStyle: {
                        bold: true,
                        color: '#bd0026',
                        italic: false
                    },
                },
            },
            hAxis: {
                // Set the title of the horizontal axis.
                title: 'Year',
                // Set the format of the numbers on the axis.
                format: 'yyyy',
                // Set the number of gridlines on the axis.
                gridlines: {
                    count: 5
                },
                // set axis boundaries
                     viewWindow: {
                       min:startDate,
                       max:endDate
                      },
                // Set the style of the text.
                titleTextStyle: {
                    bold: true,
                    italic: false
                },
            },
// Set the type of curve for the line chart.
            	curveType: 'function',
            // Set the height of the chart area.
            chartArea: {
                height: '53%'
            },
            tooltip: {
                trigger: 'none'
            },
	// Define visualization options for individual crown NDVI (series 0) 
// and daily mean NDVI (series 1)

  	series: {
   	 0: {pointSize: 1, color: 'blue', lineWidth: 0}, // Blue points for individual tree NDVI
    	1: {pointSize: 0, color: 'red', lineWidth: 1} // Red line for daily mean NDVI
 	 },
 	 interpolateNulls: true // Enable interpolation between points to connect daily mean line
});
```

[Example of chart generated in Google Earth Engine app when user clicks on an oak crown](/ee-chart(6).png)


## Python
1.	If Python (version 3.11 or higher) was not already installed, it was downloaded and installed from Python.org.
2.	If necessary, required packages were installed using the computer terminal with commands such as:

```pip install numpy 
pip install pandas 
pip install matplotlib 
```
3.	Necessary packages, including numpy, pandas, matplotlib.pyplot, and matplotlib.dates, were imported.

```# IMPORT NECESSARY PACKAGES
import numpy as np
import pandas as pd
import matplotlib.pyplot as plt
import matplotlib.dates as mdates
```

4.	Variables were created for the rolling frequency (for calculating rolling mean), z-score for the desired confidence interval for plots (1.96 was used for a 95% confidence interval), and NDVI decrease threshold to identify unhealthy trees (0.3 was used for Quercus agrifolia).

```rolling_frequency = 6  # This parameter is used for calculating rolling mean, can be changed for smoothing 
z_score = 1.96  # z-score for a 95% confidence interval (For 90%, use 1.65. For 99%, use 2.58)
ndvi_drop = 0.3  # threshold of ndvi drop to be classified as unhealthy
```

5.	The CSV file with NDVI values for each date for each tree crown, as generated in Google Earth Engine, was imported.

```# PARAMETERS (CHANGE SPECIES, SPECIES_FOR_FILES, and FILE_PATH)
# Fill in species name for plot titles in this format
species = 'Quercus agrifolia'   #coast live oak
species_for_files = 'Quercus_agrifolia'
# Set "file_path" to path to working directory 
file_path = '/Users/sarah/Downloads/'
#load csv into python using pandas package
s2 = pd.read_csv(file_path + 'S2_NDVI-' + species_for_files + '.csv', low_memory = False)
```

6.	Rows with NA values or NDVI < 0.1 were filtered out, and the date column was converted to Python format.

```# Drop rows with null values
s2 = s2.dropna()
# Drop points with NDVI < 0.1 (probably not vegetation)
s2 = s2[s2['NDVI'] > 0.1]
# Convert date to Python-compatible format
s2['date'] = pd.to_datetime(s2['date'], format='%m/%d/%Y')
```

7.	A table was created with the daily mean NDVI across all crowns, and rows with NA values or mean NDVI < 0.1 were filtered out.

```#create table with average daily NDVI across all trees
#group dataframe by date and calculate mean NDVI for each data
daily_mean = s2.groupby('date')['NDVI'].mean().reset_index()
#drop rows with null values
daily_mean = daily_mean.dropna()
#drop rows with daily mean < 0.1
daily_mean = daily_mean[daily_mean['NDVI'] > 0.1]
```

8.	Rows with daily mean NDVI < 0.1 were filtered out from the table with all tree crowns for each date. This table, s2, was then limited to only the necessary attributes for further analysis of NDVI. 

```# Filter out time series dataset for days with mean NDVI > 0.1
s2 = s2[s2['date'].isin(daily_mean['date'])]

# 2: CLEAN UP TIME SERIES DATASET ( Only keep necessary attributes relating to ndvi, geometry, id, or date )
attributes = ['system:index', 'treeID', 'NDVI', 'date', '.geo']
s2 = s2[attributes]
```

9.	All tree crowns with NDVI on the y-axis and date on the x-axis were plotted. The confidence interval and rolling mean were calculated and included in the chart, which was exported to the working directory.

```# 4: PLOT CONFIDENCE INTERVAL 
# 4.1 -- Plot confidence interval with points
# Identify the first collection date for the series
first_collection_date = s2['date'].min()
# Filter the DataFrame for the rows with the first collection date
first_collection_rows = s2[s2['date'] == first_collection_date]
# Get the number of rows, which represents the number of trees 
num_collection_points = len(first_collection_rows)


# Set window size accordingly to be "days_in_window" times the number of collection points 
window_size = num_collection_points*rolling_frequency
# Calculate rolling mean and standard deviation throughout time series //rolling mean negates the effect of outliers, data inconsistencies 
rolling_mean = pd.Series(s2['NDVI']).rolling(window_size, center=True).mean()
rolling_std = pd.Series(s2['NDVI']).rolling(window_size, center=True).std()


# Compute the upper and lower bounds of the 95% confidence interval on a rolling mean basis and plot
conf_interval = z_score * rolling_std / np.sqrt(window_size)
upper_bound = rolling_mean + conf_interval
lower_bound = rolling_mean - conf_interval
fig1, ax1 = plt.subplots()
ax1.fill_between(s2['date'], upper_bound, lower_bound, alpha=1, label='95% Confidence Interval', interpolate=True)
ax1.scatter(s2['date'], s2['NDVI'], s=0.6, alpha=0.2)
# parameter s relates to point size, alpha relates to opacity of points
ax1.set_title(species + ' NDVI (Sentinel-2)', fontsize=16)
ax1.set_xlabel('Date', fontsize=14, labelpad=10)
ax1.set_ylabel('Normalized Difference Vegetation Index (NDVI)', fontsize=14, labelpad=10)
# Change formatting of x-axis (dates) to be more readable (month year format (ex: Jan 2019))
x_axis_ticks = mdates.MonthLocator(interval=4) # have an axis tick every 4 months
ax1.xaxis.set_major_locator(x_axis_ticks)
month_year_fmt = mdates.DateFormatter('%b %Y')
ax1.xaxis.set_major_formatter(month_year_fmt)
# Set figure size to fullscreen
fig1.set_size_inches(1920/80, 1080/80)  # Assuming a screen resolution of 1920x1080 pixels
# save figure to working directory (dpi parameter relates to image quality, 300+ recommended)
fig1.savefig(file_path + species_for_files + '_CI_With_Points.png', dpi=300)
```
[Orange points represent NDVI for each oak crown on each date, and blue area represents 95% confidence interval for the 30-day rolling mean](/Quercus_agrifolia_CI_With_Points(1).png)

10.	The confidence interval and rolling mean were plotted without the points for individual tree crowns, and the plot was exported to the working directory.

```# 4.1 -- Plot confidence interval without points  
fig2, ax2 = plt.subplots()
ax2.fill_between(s2['date'], upper_bound, lower_bound, alpha=1, label='95% Confidence Interval', interpolate=True)
ax2.set_title(species + ' NDVI (Sentinel-2)', fontsize=16)
ax2.set_xlabel('Date', fontsize=14, labelpad=10)
ax2.set_ylabel('Normalized Difference Vegetation Index (NDVI)', fontsize=14, labelpad=10)
# Change formatting of x-axis (dates) to be more readable (month year format (ex: Jan 2019))
x_axis_ticks = mdates.MonthLocator(interval=4) # have an axis tick every 4 months
ax2.xaxis.set_major_locator(x_axis_ticks)
month_year_fmt = mdates.DateFormatter('%b %Y')
ax2.xaxis.set_major_formatter(month_year_fmt)
# Set figure size to fullscreen
fig2.set_size_inches(1920/80, 1080/80)  # Assuming a screen resolution of 1920x1080 pixels
# save figure to working directory
fig2.savefig(file_path + species_for_files + '_CI_Without_Points.png', dpi=300)
```
[95% confidence interval for the 30-day rolling mean](/Quercus_agrifolia_CI_Without_Points(1).png)

11.	The mean NDVI for each tree was calculated, and a figure with a boxplot showing NDVI values for each tree, with tree ID on the x-axis, was created. The x-axis was sorted from the smallest mean NDVI to the largest mean NDVI, and the plot was exported to the working directory.

``` # 5: PLOT BY TREE ID
# 5.1 -- Boxplot
# Calculate the mean NDVI for each tree(treeID)  
mean_ndvi = s2.groupby('treeID')['NDVI'].mean()
# Sort in ascending order of means
sorted_tree_ids = mean_ndvi.sort_values(ascending=True).index.tolist()
# Plot boxplot
fig3, ax3 = plt.subplots()
boxplot_dict = ax3.boxplot([s2[s2['treeID'] == tid]['NDVI'] for tid in sorted_tree_ids], labels=sorted_tree_ids)
# Customize the mean point
median_line = boxplot_dict['medians']
for line in median_line: 
    line.set(color='blue', linewidth=2)
# Customize the outliers
outlier_markers = boxplot_dict['fliers']    
for marker in outlier_markers:
    marker.set(marker='o', markerfacecolor='red', markersize=2, markeredgecolor='none')
#set title and axis labels
ax3.set_title(species + ' NDVI (Sentinel-2)', fontsize=16)
ax3.set_xlabel('Tree ID', fontsize=14, labelpad=10)
ax3.set_ylabel('Normalized Difference Vegetation Index (NDVI)', fontsize=14, labelpad=10)
# Remove x-axis tick labels for neatness
ax3.set_xticklabels([])
# Set figure size to fullscreen
fig3.set_size_inches(1920/80, 1080/80)  # Assuming a screen resolution of 1920x1080 pixels
# save figure to working directory
fig3.savefig(file_path + species_for_files + '_Boxplot', dpi=300)
```
[Boxplot of NDVI values across study period for each oak crown](/Quercus_agrifolia_Boxplot(1).png)

12.	Using a for loop, trees with NDVI drops above the specified threshold were identified. From the table with all tree crowns, only the identified trees were selected, and a new table was created from this selection. The new table was exported to the working directory. 

``` # 5.2 -- Identify Trees That Have Had Significant Drop in NDVI
trees_by_id = s2.groupby('treeID')
# List to store tree IDs with an NDVI drop of 0.3 or greater  
tree_ids_dropped_NDVI = []
# Loop through each tree's collection points
for tree_id, tree in trees_by_id:
    # If NDVI drop is 0.3 or greater, add to list   
    if any(tree['NDVI'].diff() <= -ndvi_drop):
        tree_ids_dropped_NDVI.append(tree_id)
#create new table from s2, subsetting only trees with NDVI drop
trees_dropped_NDVI = s2[s2['treeID'].isin(tree_ids_dropped_NDVI)]   
# Export these trees Sentinel-2 data into csv for further analysis with GIS
trees_dropped_NDVI.to_csv(file_path + species_for_files + '_dropped_ndvi.csv')
```

13.	The rolling mean and confidence interval for the daily mean NDVI table were calculated, and the upper and lower bounds of the confidence interval were used to identify high and low outliers. A scatterplot of daily mean NDVI values, with the confidence interval, rolling mean, high outliers, and low outliers differentiated by point color, was created. The plot was exported to the working directory.

``` # 6: PLOT DAILY MEANS 
# 6.1 -- Confidence Interval
rolling_mean = pd.Series(daily_mean['NDVI']).rolling(rolling_frequency, center=True).mean()
rolling_std = pd.Series(daily_mean['NDVI']).rolling(rolling_frequency, center=True).std()
# Compute the upper and lower bounds of the 95% confidence interval
conf_interval = z_score * rolling_std / np.sqrt(rolling_frequency)
upper_bound = rolling_mean + conf_interval
lower_bound = rolling_mean - conf_interval
# Identify outliers
outliers_high_indices = np.where(daily_mean['NDVI'] > upper_bound)[0]
outliers_low_indices = np.where(daily_mean['NDVI'] < lower_bound)[0]
outliers_high = pd.DataFrame({'date': daily_mean['date'].iloc[outliers_high_indices],
                              'NDVI': daily_mean['NDVI'].iloc[outliers_high_indices]})

outliers_low = pd.DataFrame({'date': daily_mean['date'].iloc[outliers_low_indices],
                             'NDVI': daily_mean['NDVI'].iloc[outliers_low_indices]})
# Create scatterplot of daily mean, differentiating points by non-outlier, high outlier, and low outlier
fig4, ax4 = plt.subplots()
ax4.scatter(daily_mean['date'], daily_mean['NDVI'], color='blue', label='Non-Outliers', s=10)
ax4.scatter(outliers_high['date'], outliers_high['NDVI'], color='red', label='Outliers (High)', s=10)
ax4.scatter(outliers_low['date'], outliers_low['NDVI'], color='darkred', label='Outliers (Low)', s=10)
# add confidence interval to plot, and legend
ax4.fill_between(daily_mean['date'], upper_bound, lower_bound, alpha=0.2, label='95% Confidence Interval', interpolate=True)
ax4.legend()
# add title and axis labels to plot
ax4.set_title(species + ' Daily Mean NDVI (Sentinel-2)', fontsize=16)
ax4.set_xlabel('Date', fontsize=14, labelpad=10)
ax4.set_ylabel('Normalized Difference Vegetation Index (NDVI)', fontsize=14, labelpad=10)
x_axis_ticks = mdates.MonthLocator(interval=4)  # have an axis tick every 4 months
ax4.xaxis.set_major_locator(x_axis_ticks)
month_year_fmt = mdates.DateFormatter('%b %Y')
ax4.xaxis.set_major_formatter(month_year_fmt)
fig4.set_size_inches(1920/80, 1080/80)  # Assuming a screen resolution of 1920x1080 pixels
# export figure to working directory
fig4.savefig(file_path + species_for_files + '_means_CI.png', dpi=300)
```
[B95% confidence interval for daily mean NDVI across all oak crowns, with each point representing a daily mean](/Quercus_agrifolia_means_CI(1).png)


# Implementation

The interactive application runs on the computational power of Google Earth Engine, a cloud-based platform for planetary-scale environmental data analysis. Anyone with the link can view and interact with this application. To implement the code in Google Earth Engine, one must create a free Google Earth Engine account and obtain shapefiles for the neighborhood boundary and tree crowns. To implement the code in Python, one must download Python as instructed and run the first script in Google Earth Engine to obtain the necessary csv file for input. 

# Known Issues

•	If a user clicks on a tree that is not Quercus agrifolia, or a point that is not a tree, an error will come up in the place of the NDVI chart.
•	Each time the user clicks on the map, the Quercus agrifolia crowns and Altadena neighborhood boundary will be added to the map again. All layers on the map can be reset by clicking on the “Reset all layers” button. 
•	Occasionally, the app will produce a ‘JSON’ error message when it is initially loaded. Reloading the page should fix this error. 

# Benefits and Applications

This GEE application offers numerous benefits, including:

•	Enhanced Monitoring and Management: Facilitates the monitoring of urban tree health and biodiversity, enabling more informed management decisions.
•	Educational Tool: Serves as an educational resource for schools and the public, raising awareness of the importance of urban forests.
•	Research and Development: Provides a valuable dataset for academic research on urban ecology and urban planning.
•	Policy and Planning: Assists policymakers in developing strategies for urban forest expansion, maintenance, and sustainability.
Implementing the code in Python will provide the user with additional information and visualizations, including:
•	Rolling mean NDVI across all crowns and 95% confidence interval for rolling mean
•	Time series showing all individual crowns
•	Time series showing 95% confidence interval for rolling mean and identifying high and low outliers
•	Table with a subset of vulnerable crowns, identified based on decreasing NDVI

# Acknowledgements
This workflow was developed by Dr. Jonathan Pando Ocón, Assistant Professor of Remote Sensing, CSU Long Beach, with Mr. Corey DeLisle, Geography, UCLA, contributing to field work and data processing, and Ms. Kristi Le, Computational Biology, UCLA, contributing to developing code and time series analysis. Many individuals were instrumental in this research, including Dr. Thomas W. Gillespie, Professor of Geography, UCLA, Dr. Elsa M. Ordway, Assistant Professor of Ecology & Evolutionary Biology, UCLA, Dr. E. Natasha Stavros, Ball Aerospace, Dr. Steven J. Steinberg, Geographic Information Officer (GIO) for Los Angeles County California and Adjunct Professor with the MS GIScience program at CSU Long Beach, and Mr. Justin Robertson, Senior Planner at Los Angeles County Department of Public Health. Mr. Andrew Niehaus, Geographic Information Science M.S., Clark University, and Ms. Sarah Hughes, Geographic Information Science M.S., Clark University, provided additional assistance with coding and implementing the interactive map application. 

# References
Alonzo, M., Bookhagen, B. and Roberts, D.A., 2014. Urban tree species mapping using hyperspectral and LiDAR data fusion. Remote Sensing of Environment, 148, pp.70-83.
Alonzo, M., McFadden, J.P., Nowak, D.J. and Roberts, D.A., 2016. Mapping urban forest structure and function using hyperspectral imagery and LiDAR data. Urban forestry & urban greening, 17, pp.135-147.
Alonzo, M., Roth, K. and Roberts, D., 2013. Identifying Santa Barbara's urban tree species from AVIRIS imagery using canonical discriminant analysis. Remote Sensing Letters, 4(5), pp.513-521.
Anderson, C.B., 2018. The CCB-ID approach to tree species mapping with airborne imaging spectroscopy. PeerJ, 6, p.e5666.
Audebert, N., Le Saux, B. and Lefèvre, S., 2018. Beyond RGB: Very high resolution urban remote sensing with multimodal deep networks. ISPRS Journal of Photogrammetry and Remote Sensing, 140, pp.20-32.
Aval, J., Demuynck, J., Zenou, E., Fabre, S., Sheeren, D., Fauvel, M., Adeline, K. and Briottet, X., 2018. Detection of individual trees in urban alignment from airborne data and contextual information: A marked point process approach. ISPRS Journal of Photogrammetry and Remote Sensing, 146, pp.197-210.
Avolio, M.L., Pataki, D.E., Gillespie, T.W., Jenerette, G.D., McCarthy, H.R., Pincetl, S. and Weller Clarke, L., 2015. Tree diversity in southern California's urban forest: the interacting roles of social and environmental variables. Frontiers in Ecology and Evolution, 3, p.73.
Belgiu, M. and Drăguţ, L., 2016. Random forest in remote sensing: A review of applications and future directions. ISPRS Journal of Photogrammetry and Remote Sensing, 114, pp.24-31.
Bioucas-Dias, J.M., Plaza, A., Dobigeon, N., Parente, M., Du, Q., Gader, P. and Chanussot, J., 2012. Hyperspectral unmixing overview: Geometrical, statistical, and sparse regression-based approaches. IEEE Journal of Selected Topics in Applied Earth Observations and Remote Sensing, 5(2), pp.354-379.
Bodnaruk, E.W., Kroll, C.N., Yang, Y., Hirabayashi, S., Nowak, D.J. and Endreny, T.A., 2017. Where to plant urban trees? A spatially explicit methodology to explore ecosystem service tradeoffs. Landscape and Urban Planning, 157, pp.457-467.
Bradley, B.A., 2014. Remote detection of invasive plants: a review of spectral, textural and phenological approaches. Biological Invasions, 16(7), pp.1411-1425.
Branson, S., Wegner, J.D., Hall, D., Lang, N., Schindler, K. and Perona, P., 2018. From Google Maps to a fine-grained catalog of street trees. ISPRS Journal of Photogrammetry and Remote Sensing, 135, pp.13-30.
Degerickx, J., Roberts, D.A., McFadden, J.P., Hermy, M. and Somers, B., 2018. Urban tree health assessment using airborne hyperspectral and LiDAR imagery. Journal of Selected Topics in Applied Earth Observations and Remote Sensing, 73, pp.26-38.
Dian, Y., Pang, Y., Dong, Y. and Li, Z., 2016. Urban tree species mapping using airborne LiDAR and hyperspectral data. Journal of the Indian Society of Remote Sensing, 44(4), pp.595-603.
Elgendy, M., 2020. Deep learning for vision systems. Simon and Schuster.
Fang, F., McNeil, B.E., Warner, T.A., Maxwell, A.E., Dahle, G.A., Eutsler, E. and Li, J., 2020. Discriminating tree species at different taxonomic levels using multi-temporal WorldView-3 imagery in Washington DC, USA. Remote Sensing of Environment, 246, pp.111811.
Fassnacht, F.E., Neumann, C., Förster, M., Buddenbaum, H., Ghosh, A., Clasen, A., Joshi, P.K. and Koch, B., 2014. Comparison of feature reduction algorithms for classifying tree species with hyperspectral data on three central European test sites. IEEE Journal of Selected Topics in Applied Earth Observations and Remote Sensing, 7(6), pp.2547-2561.
Fassnacht, F.E., Latifi, H., Stereńczak, K., Modzelewska, A., Lefsky, M., Waser, L.T., Straub, C. and Ghosh, A., 2016. Review of studies on tree species classification from remotely sensed data. Remote Sensing of Environment, 186, pp.64-87.
Ferreira, M.P., Zortea, M., Zanotta, D.C., Shimabukuro, Y.E. and de Souza Filho, C.R., 2016. Mapping tree species in tropical seasonal semi-deciduous forests with hyperspectral and multispectral data. Remote Sensing of Environment, 179, pp.66-78.
Fricker, G.A., Ventura, J.D., Wolf, J.A., North, M.P., Davis, F.W. and Franklin, J., 2019. A convolutional neural network classifier identifies tree species in mixed-conifer forest from hyperspectral imagery. Remote Sensing, 11(19), p.2326.
Gillespie, T.W., de Goede, J., Aguilar, L., Jenerette, G.D., Fricker, G.A., Avolio, M.L., Pincetl, S., Johnston, T., Clarke, L.W. and Pataki, D.E., 2017. Predicting tree species richness in urban forests. Urban Ecosystems, 20(4), pp.839-849.
Hartling, S., Sagan, V., Sidike, P., Maimaitijiang, M. and Carron, J., 2019. Urban tree species classification using a WorldView-2/3 and LiDAR data fusion approach and deep learning. Sensors, 19(6), p.1284.
Hughes, G., 1968. On the mean accuracy of statistical pattern recognizers. IEEE transactions on information theory, 14(1), pp.55-63.
Immitzer, M., Atzberger, C. and Koukal, T., 2012. Tree species classification with random forest using very high spatial resolution 8-band WorldView-2 satellite data. Remote Sensing, 4(9), pp.2661-2693.
Immitzer, M., Vuolo, F. and Atzberger, C., 2016. First experience with Sentinel-2 data for crop and tree species classifications in central Europe. Remote Sensing, 8(3), p.166.
Jensen, R.R., Hardin, P.J. and Hardin, A.J., 2012. Classification of urban tree species using hyperspectral imagery. Geocarto International, 27(5), pp.443-458.
Jombo, S., Adam, E. and Odindi, J., 2021. Classification of tree species in a heterogeneous urban environment using object-based ensemble analysis and World View-2 satellite imagery. Applied Geomatics, 13(3), pp.373-387.
Jombo, S., Adam, E. and Tesfamichael, S., 2022. Classification of urban tree species using LiDAR data and WorldView-2 satellite imagery in a heterogeneous environment. Geocarto International, 37(25), pp.9943-9966.
Li, H., Lee, W.S., Wang, K., Ehsani, R. and Yang, C., 2014. ‘Extended spectral angle mapping (ESAM)’for citrus greening disease detection using airborne hyperspectral imaging. Precision Agriculture, 15(2), pp.162-183.
Li, W., Liu, H., Wang, Y., Li, Z., Jia, Y. and Gui, G., 2019. Deep learning-based classification methods for remote sensing images in urban built-up areas. Ieee Access, 7, pp.36274-36284.
Lin, Y. and Hyyppä, J., 2016. A comprehensive but efficient framework of proposing and validating feature parameters from airborne LiDAR data for tree species classification. International journal of applied earth observation and geoinformation, 46, pp.45-55.
Liu, L., Coops, N.C., Aven, N.W. and Pang, Y., 2017. Mapping urban tree species using integrated airborne hyperspectral and LiDAR remote sensing data. Remote Sensing of Environment, 200, pp.170-182.
Love, N.L., Nguyen, V., Pawlak, C., Pineda, A., Reimer, J.L., Yost, J.M., Fricker, G.A., Ventura, J.D., Doremus, J.M., Crow, T. and Ritter, M.K., 2022. Diversity and structure in California’s urban forest: What over six million data points tell us about one of the world's largest urban forests. Urban Forestry & Urban Greening, 74, p.127679.
Ma, Q., Lin, J., Ju, Y., Li, W., Liang, L. and Guo, Q., 2023. Individual structure mapping over six million trees for New York City USA. Scientific Data, 10(1), p.102.
Marrs, J. and Ni-Meister, W., 2019. Machine learning techniques for tree species classification using co-registered LiDAR and hyperspectral data. Remote Sensing, 11(7), p.819.
Martins, G.B., La Rosa, L.E.C., Happ, P.N., Coelho Filho, L.C.T., Santos, C.J.F., Feitosa, R.Q. and Ferreira, M.P., 2021. Deep learning-based tree species mapping in a highly diverse tropical urban setting. Urban Forestry & Urban Greening, 64, p.127241.
Mesquita, M.R., Agarwal, S., de Morais Lima, L.H.G., Soares, M.R.A., Barbosa, D.B.E.S., Silva, V.C., Werneck, G.L. and Costa, C.H.N., 2022. The use of geotechnologies for the identification of the urban flora in the city of Teresina, Brazil. Urban Ecosystems, pp.1-12.
Neyns, R. and Canters, F., 2022. Mapping of urban vegetation with high-resolution remote sensing: A review. Remote sensing, 14(4), p.1031.
Ossola, A., Hoeppner, M.J., Burley, H.M., Gallagher, R.V., Beaumont, L.J. and Leishman, M.R., 2020. The Global Urban Tree Inventory: A database of the diverse tree flora that inhabits the world’s cities. Global Ecology and Biogeography, 29(11), pp.1907-1914.
Pleșoianu, A.I., Stupariu, M.S., Șandric, I., Pătru-Stupariu, I. and Drăguț, L., 2020. Individual tree-crown detection and species classification in very high-resolution remote sensing imagery using a deep learning ensemble model. Remote Sensing, 12(15), p.2426.
Pretzsch, H., Biber, P., Uhl, E., Dahlhausen, J., Schütze, G., Perkins, D., Rötzer, T., Caldentey, J., Koike, T., Con, T.V. and Chavanne, A., 2017. Climate change accelerates growth of urban trees in metropolises worldwide. Scientific reports, 7(1), pp.1-10.
Pu, R. and Liu, D., 2011. Segmented canonical discriminant analysis of in situ hyperspectral data for identifying 13 urban tree species. International Journal of Remote Sensing, 32(8), pp.2207-2226.
Pu, R. and Landry, S., 2012. A comparative analysis of high spatial resolution IKONOS and WorldView-2 imagery for mapping urban tree species. Remote Sensing of Environment, 124, pp.516-533.
Pu, R., Landry, S. and Yu, Q., 2018. Assessing the potential of multi-seasonal high resolution Pléiades satellite imagery for mapping urban tree species. International Journal of Applied Earth Observation and Geoinformation, 71, pp.144-158.
Pu, R., 2021. Mapping tree species using advanced remote sensing technologies: a state-of-the-art review and perspective. Journal of Remote Sensing, 2021, p.26.
Stereńczak, K., Laurin, G.V., Chirici, G., Coomes, D.A., Dalponte, M., Latifi, H. and Puletti, N., 2020. Global airborne laser scanning data providers database (GlobALS)—A new tool for monitoring ecosystems and biodiversity. Remote Sensing, 12(11), p.1877.
Stoker, J. and Miller, B., 2022. The accuracy and consistency of 3D elevation program data: a systematic analysis. Remote Sensing, 14(4), p.940.
Wang, Z., Fan, C. and Xian, M., 2021. Application and evaluation of a deep learning architecture to urban tree canopy mapping. Remote Sensing, 13(9), p.1749.
Waters, E., Oghaz, M.M. and Saheer, L.B., 2021. Urban tree species classification using aerial imagery. arXiv preprint arXiv:2107.03182.
Yang, G., Zhao, Y., Li, B., Ma, Y., Li, R., Jing, J. and Dian, Y., 2019. Tree species classification by employing multiple features acquired from integrated sensors. Journal of Sensors, 2019.
Yang M, Zhou X, Liu Z, Li P, Tang J, Xie B, Peng C. A review of general methods for quantifying and estimating urban trees and biomass. Forests. 2022 13(4). p.616.
Zhang, C. and Qiu, F., 2012. Mapping individual tree species in an urban forest using airborne LiDAR data and hyperspectral imagery. Photogrammetric Engineering & Remote Sensing, 78(10), pp.1079-1087.
Zhang, C., Sargent, I., Pan, X., Li, H., Gardiner, A., Hare, J. and Atkinson, P.M., 2018. An object-based convolutional neural network (OCNN) for urban land use classification. Remote sensing of environment, 216, pp.57-70.
Zhang, R., Li, Q., Duan, K.F., You, S.C., Zhang, T., Liu, K. and Gan, Y.H., 2020. PRECISE CLASSIFICATION OF FOREST SPECIES BASED ON MULTI-SOURCE REMOTE-SENSING IMAGES. APPLIED ECOLOGY AND ENVIRONMENTAL RESEARCH, 18(2), pp.3659-3681.
Zhou, J., Qin, J., Gao, K. and Leng, H., 2016. SVM-based soft classification of urban tree species using very high-spatial resolution remote-sensing imagery. International Journal of Remote Sensing, 37(11), pp.2541-2559.


