# corridor-mapping
Mapping the corridors between the protected areas in North-Bengal
//This script can be used for mapping-classifying the landscape of a specific region using field points
//Author: Dipanwita Dutta
//Date: 2023-2024

//Import Image, Mask Clouds and Median Value
function maskS2clouds(image){
  var qa = image.select('QA60');

// Bits 10 and 11 are clouds and cirrus, respectively.
  var cloudBitMask = 1 << 10;
  var cirrusBitMask = 1 << 11;

// Both flags should be set to zero, indicating clear conditions.
  var mask = qa.bitwiseAnd(cloudBitMask).eq(0)
      .and(qa.bitwiseAnd(cirrusBitMask).eq(0));

  return image.updateMask(mask).divide(10000);
}
//var corridorname_buffer = corridorname.geometry().buffer(7800);

var Sentinel_S2_SR = ee.ImageCollection('COPERNICUS/S2_SR')
                    .filterDate('start date', 'end date')
                    .filter(ee.Filter.lt('CLOUDY_PIXEL_PERCENTAGE', 5))
                    .filterBounds(corridorname)
                    .map(maskS2clouds)
                    .median();
                    
// Map.addLayer(Sentinel_S2_SR.clip(corridorname_buffer), {min: 0.0, max: 1.0, bands: ['B8', 'B4', 'B3'], gamma: [0.95, 1.1, 1]}, 'RGB');

//DEM
var Elevation = ee.Image("USGS/GMTED2010").select('be75').clip(corridorname);
var be75 = Elevation.resample('bilinear').reproject({crs: Elevation.projection(),scale: 10}).divide(1000);
//Map.addLayer(be75)
var ndvi = Sentinel_S2_SR.normalizedDifference(['B8','B4'])
Map.addLayer(ndvi)
var Sentinel_S2_SR = Sentinel_S2_SR.addBands(be75).addBands(ndvi).clip(CR1);
Map.addLayer(Sentinel_S2_SR, {min: 0.3, max: 0.7, bands: ['B8', 'B4', 'B3'], gamma: [0.95, 1.1, 1]}, 'RGB_ELE');//

//Creating Training Points
var Training = class.merge(class).merge(class).merge(class); //class signifies the lulc type/name of the specific lulc defined by the user

//Creating Training Data
var S2bands = ['B2', 'B3', 'B4', 'B8', 'B8A', 'B11', 'B12', 'be75','nd']; //select bands for classification
var training_corridor = Sentinel_S2_SR.select(S2bands).sampleRegions({ //extract bandwise pixel values
  collection: Training,
  properties: ['given by the user'], 
  scale: 10 //spatial resolution
});

//Training the classifier
var classifier_RF_2022 = ee.Classifier.smileRandomForest(10).train({
  features: training_corridor, 
  classProperty: 'given by the user', 
  inputProperties: S2bands
});

//Run Classifier
var Corridor_classified = Sentinel_S2_SR.select(S2bands).classify(classifier_RF_2022).clip(corridorname).focalMedian(2);

//View Classified Image
Map.addLayer(Corridor_classified, {min: ..., max: ..., palette: ['FAED29', '8505F0', 'FC24F9','B9B4B9','F90F3D','F9760F',
                                                                '0F8E1C','49EE35']}, 'Corridor_LULC');
//,'0D21E9','58595B','F8980D','F86DAA','66099C','34BA5B'
