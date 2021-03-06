---
title: "Temporal and Spatial Reducers"
teaching: 0
exercises: 0
questions:
- How do I aggregate a time series of raster data over a time period?
- How do I summarize data by vector regions?
- How do I export tabular data summaries?
objectives:
- Use reducers to aggregate a daily image collection to annual values
- Import vector data to summarize values by polygon regions
- Use climate data products available through GEE
- Export tabular data
keypoints:
- GEE hosts a wide variety of useful spatial datasets
- Reducers aggregate or summarize data in space and time
- There are several ways to use vector data in GEE
- Results can be exported to Google Drive or Google Cloud
---

Link to a static version of the full script used in this module:
[https://code.earthengine.google.com/d3c55e2958b9ae1e0bb6fe3c5e9da066](https://code.earthengine.google.com/d3c55e2958b9ae1e0bb6fe3c5e9da066)

## Reducers: Overview

In Google Earth Engine (GEE), [reducers](https://developers.google.com/earth-engine/reducers_intro) are used to aggregate data over time, space, and other data structures. They belong to the `ee.Reducer` class and include summary statistics, histograms, and linear regression, among others. Here's a diagram from Google demonstrating a reducer applied to an `ImageCollection`:

<img src="../fig/GEE_Reduce_ImageCollection.png" border = "10">

Reductions can also occur in space, over bands within an image, or over the attributes of a `FeatureCollection`. See the [Reducer Overview](https://developers.google.com/earth-engine/reducers_intro) in the Google Developer's Guide for more information.

## Exercise: Obtain climate data from GEE
Here, we will demonstrate a temporal reducer and a spatial reducer by obtaining data on annual precipitation by US county.

### GEE Data Catalog
A secondary objective to this exercise is to use GEE to access common datasets stored in the data archive that may appeal to those not directly interested in remote sensing applications. As described in the [Introduction](https://geohackweek.github.io/GoogleEarthEngine/01-introduction/), GEE has co-located a number of datasets relevant to earth systems analyses. The full archive can be browsed [here](https://code.earthengine.google.com/datasets/). In this exercise, we will use the [GRIDMET Meteorological Dataset](https://code.earthengine.google.com/dataset/IDAHO_EPSCOR/GRIDMET) to obtain precipitation. Briefly, GRIDMET blends PRISM and NLDAS to produce a daily, 4 km gridded climate dataset for the contiguous United States from 1979 - present.

### Part 1: Reduce an ImageCollection to Aggregate Over Time
As discussed in [Accessing Satellite Imagery](https://geohackweek.github.io/GoogleEarthEngine/03-load-imagery/), an `ImageCollection` is a stack or time series of images. Reducers are used to derive a single `Image` based on the `ImageCollection`. Operations occur on a per pixel basis.

**Processing Overview**

* "Load" the GRIDMET data as an `ImageCollection`
* Filter for the precipitation data band and dates desired (2016)
* **Reduce** 365 "raster" images of daily precipitation into one raster image of annual precipitation totals (aka sum rasters by pixel)
* Visualize the result

#### Get and Filter the ImageCollection
First, we need to identify the **ImageCollection ID** for the GRIDMET data product and the **band name** for the precipitation data (and check any relevant metadata). You can find this either in the [data catalog](https://code.earthengine.google.com/datasets/) or directly in the [GEE Code Editor](https://code.earthengine.google.com/) at the top above  the center panel.

From the [GRIDMET description](https://code.earthengine.google.com/dataset/IDAHO_EPSCOR/GRIDMET), we know the ImageCollection ID = 'IDAHO_EPSCOR/GRIDMET' and the precipitation band name is 'pr'. We will specifically `select` this band only.

{% highlight javascript %}
// load precip data (mm, daily total): 365 images per year
var cPrecip = ee.ImageCollection('IDAHO_EPSCOR/GRIDMET')
                    .select('pr')   // select  precip band only
                    .filterDate('2016-01-01', '2016-12-31');
print(cPrecip);  
{% endhighlight %}


By printing the resulting collection to the Console, we can see we've accessed 365 images, each with 1 band named 'pr'.

#### Apply a Sum Reducer and Visualize Results
The `imageCollection.reduce()` operator allows you to apply any function of class `ee.Reducer()` to all images in the collection. If your `ImageCollection` had multiple bands, the reducer is applied separately to all bands (unless the reducer uses multiple bands as inputs, in which case the number of bands in the image collection must match the number of inputs required by the reducer). You can find available reducers and their descriptions in the searchable API reference under the **Docs** tab in the upper left panel of the code editor.

<br>
<img src="../fig/05_reducerMenu.PNG" border = "10">
<br><br>

Some commonly used reducers have shortcut syntax, such as `imageCollection.mean()`, `imageCollection.min()`, and conveniently, `imageCollection.sum()`. Both syntaxes are demonstrated in the following code chunk.

{% highlight javascript %}
// reduce the image collection to one image by summing the 365 daily rasters
var annualPrecip = cPrecip.reduce(ee.Reducer.sum());
print(annualPrecip);

// using equivalent "shortcut" notation available for some common stats:
var annualPrecip = cPrecip.sum();

// visualize annual precipitation --------------------------------------
var precipPal = ['white','blue'] // store palette as variable
Map.addLayer(annualPrecip, {min: 0, max: 3000, palette: precipPal}, 'precip');
{% endhighlight %}

By printing the resulting image to the Console, we can see we now have 1 image with 1 band named 'pr_sum'. Here's what it looks like:

<br>
<img src="../fig/05_annualPrecipMap.PNG" border = "10">
<br><br>

### Part 2: Spatial Reducer: Get Image Statistics By Regions
The objective of Part 2 is to take the image of annual precipitation we just created and get the mean annual precipitation by county in the United States. To get image statistics for multiple regions, we can use an [image.reduceRegions()](https://developers.google.com/earth-engine/reducers_reduce_regions) call. We will use a [FeatureCollection](https://developers.google.com/earth-engine/feature_collections) to store our vector dataset of counties. Note that there is also a [image.reduceRegion()](https://developers.google.com/earth-engine/reducers_reduce_region) operator if you wanted to summarize one polygon region only. The result of the `reduceRegions()` operation is added to the properties of each feature in the `FeatureCollection`.

**An important note on the scale parameter**

GEE uses lazy code evaluation that only executes parts of your script needed for results - in the case of the JavaScript API code editor environment, that means things needed to fulfill print statements, map visualizations, or export tasks. *GEE will run your computations at the resolution of your current map view in the code editor unless you tell it otherwise.* Whenever possible, explicitly set the scale arguments to force GEE to work in a scale that makes sense for your imagery/analysis.

#### Load the County Boundaries (Vector Data)
There are three ways to use vector data in GEE:

* [Upload a shapefile](https://developers.google.com/earth-engine/importing) to your personal *Asset* folder in the top left panel. You can set sharing permissions on these as needed. We use an asset vector file in the [Accessing Satellite Imagery module](https://geohackweek.github.io/GoogleEarthEngine/03-load-imagery/).
* Import an existing [Google Fusion Table](https://support.google.com/fusiontables#topic=1652595), or [create your own](https://fusiontables.google.com/data?dsrcid=implicit) fusion table from a KML in WGS84.  Each fusion table has a unique Id (File > About this table) that can be used to load it into GEE. GEE only recently added the Asset option, so you may see folks still using fusion tables in the forums, etc. If you have the choice, I'd use an asset.
* Manually draw points, lines, and polygons using the geometry tools in the code editor. We do this in the [Classify Imagery Module](https://geohackweek.github.io/GoogleEarthEngine/05-classify-imagery/).

Here, we will use an [existing public fusion table of county boundaries](https://fusiontables.google.com/data?docid=1xdysxZ94uUFIit9eXmnw1fYc6VcQiXhceFd_CVKa#map:id=2) from the US Census Bureau.

This dataset includes entities outside of the contiguous US such as Alaska, Puerto Rico, and American Samoa. We will remove these based on their unique ID's in a property attribute containing "state" FIPS codes to demonstrate vector filtering.

{% highlight javascript %}
// load regions: counties from a public fusion table, removing non-conus states
// by using a custom filter
var nonCONUS = [2,15,60,66,69,72,78] // state FIPS codes that we don't want
var counties = ee.FeatureCollection('ft:1ZMnPbFshUI3qbk9XE0H7t1N5CjsEGyl8lZfWfVn4')
        .filter(ee.Filter.inList('STATEFP',nonCONUS).not());
print(counties);

// visualize
Map.addLayer(counties,{},'counties');  
{% endhighlight %}

By printing the county featureCollection, we see there are 3108 county polygons and 11 columns of attribute data.

<br>
<img src="../fig/05_countyMap.png" border = "10">
<br><br>

#### Apply the spatial reducer

{% highlight javascript %}
// get mean precipitation values by county polygon
var countyPrecip = annualPrecip.reduceRegions({
  collection: counties,
  reducer: ee.Reducer.mean(),
  scale: 4000 // the resolution of the GRIDMET dataset
});
print(countyPrecip);
{% endhighlight %}

By printing the countyPrecip featureCollection, we see there are 3108 county polygons and now 12 columns of attribute data, with the addition of the "mean" column.

## Format and Export Results
GEE can export tables in CSV (default), GeoJSON, KML, or KMZ. Here, we do a little formatting to prepare our FeatureCollection for export as a CSV.

Formatting includes:

* removing the .geo column for a tidier dataset (this column can get quite large when polygons are highly detailed, but there's no reason that you have to do this step)
* Adding a Column Attribute for the year of the precip data to demonstrate attribute manipulation. This can only be done to Features, so we map a function to do this over the features within the Feature Collection

{% highlight javascript %}
// drop .geo column
var polyOut = countyPrecip.select(['.*'],null,false);

// add a new column for year
polyOut = polyOut.map(function(f){
  return f.set('Year',2016);
});

// Export ---------------------------------------------------------------------
// An example for how to export
Export.table.toDrive({
  collection: polyOut,
  description: 'GRIDMET_annual_precip_by_county',
  folder: 'GEE_geohackweek',
  fileFormat: 'CSV'
});   

// AND HIT 'RUN' IN THE TASKS TAB IN THE UPPER RIGHT PANEL
{% endhighlight %}

Note on the folder name: If this folder exists within  your Google Drive, GEE will find it and export here regardless of the full file path for the folder. If the folder doesn't exist, GEE will create it upon export.

**Final Step: Start the Export Task**

In order to actually export your data, you have to explicitly hit the "Run" button under the "Tasks" tab in the upper right panel of the code editor. It should take 20-30 seconds to export, depending on GEE user loads.

<br>
<img src="../fig/05_runTask.png" border = "10">
<br><br>

Link to a static version of the full script used in this module:
(https://code.earthengine.google.com/d3c55e2958b9ae1e0bb6fe3c5e9da066)
