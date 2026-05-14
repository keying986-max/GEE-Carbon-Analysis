# GEE NPP Trend Analysis
/***************************************************************
 * NPP Trend and Stability Analysis Based on MODIS MOD17A2HGF
 *
 * Platform:
 *   Google Earth Engine Code Editor
 *
 * Dataset:
 *   MODIS/061/MOD17A2HGF
 *
 * Band:
 *   PsnNet
 *
 * Method:
 *   1. Calculate annual NPP by summing 8-day PsnNet images.
 *   2. Estimate the linear trend slope of annual NPP.
 *   3. Classify trend significance using Pearson correlation p-value.
 *   4. Classify ecosystem stability using coefficient of variation.
 *
 * Outputs:
 *   1. S
 *      Linear trend slope of annual NPP.
 *
 *   2. CLASS
 *      Trend significance classification:
 *        1 = Significant increase, S > 0 and p < 0.05
 *        2 = Significant decrease, S < 0 and p < 0.05
 *        3 = Non-significant or insufficient data
 *
 *   3. CV_CLASS
 *      Stability classification based on coefficient of variation:
 *        1 = CV < 0.1
 *        2 = 0.1 <= CV < 0.2
 *        3 = 0.2 <= CV < 0.3
 *        4 = CV >= 0.3
 *
 * Notes:
 *   Please modify the CONFIG section before running this script.
 ***************************************************************/


/***************************************************************
 * 1. User configuration
 ***************************************************************/

var CONFIG = {
  // ----------------------------------------------------------------
  // Study area
  // Replace this with your own GEE asset path.
  // Example format: 'users/your_username/your_asset_name'
  // ----------------------------------------------------------------
  studyAreaAsset: 'users/your_username/your_asset_name',

  // ----------------------------------------------------------------
  // Analysis period
  // Fill in the start and end year before running.
  // Both values should be numbers.
  // ----------------------------------------------------------------
  startYear: null,
  endYear: null,

  // ----------------------------------------------------------------
  // Dataset settings
  // This script is designed for MODIS MOD17A2HGF NPP analysis.
  // ----------------------------------------------------------------
  datasetId: 'MODIS/061/MOD17A2HGF',
  nppBand: 'PsnNet',
  scaleFactor: 0.0001,

  // ----------------------------------------------------------------
  // Statistical settings
  // ----------------------------------------------------------------
  minSamples: 3,
  significanceLevel: 0.05,

  // ----------------------------------------------------------------
  // Export settings
  // ----------------------------------------------------------------
  exportFolder: 'GEE_NPP_Trend',
  exportCrs: 'EPSG:4326',
  exportScale: 500,
  maxPixels: 1e13,

  // ----------------------------------------------------------------
  // Output name prefix
  // ----------------------------------------------------------------
  outputPrefix: 'npp',

  // ----------------------------------------------------------------
  // Map display
  // ----------------------------------------------------------------
  mapZoom: 6
};


/***************************************************************
 * 2. Configuration check
 ***************************************************************/

function validateConfig(config) {
  var placeholderAsset = 'users/your_username/your_asset_name';

  if (!config.studyAreaAsset || config.studyAreaAsset === placeholderAsset) {
    throw new Error(
      'Please set CONFIG.studyAreaAsset to your own Google Earth Engine asset path.'
    );
  }

  if (config.startYear === null || config.endYear === null) {
    throw new Error(
      'Please set CONFIG.startYear and CONFIG.endYear before running the script.'
    );
  }

  if (typeof config.startYear !== 'number' || typeof config.endYear !== 'number') {
    throw new Error(
      'CONFIG.startYear and CONFIG.endYear should be numbers.'
    );
  }

  if (config.startYear > config.endYear) {
    throw new Error(
      'CONFIG.startYear should not be greater than CONFIG.endYear.'
    );
  }

  var totalYears = config.endYear - config.startYear + 1;

  if (totalYears < config.minSamples) {
    throw new Error(
      'The analysis period is too short. It should contain at least ' +
      config.minSamples +
      ' years.'
    );
  }
}

validateConfig(CONFIG);


/***************************************************************
 * 3. Basic variables
 ***************************************************************/

var studyArea = ee.FeatureCollection(CONFIG.studyAreaAsset).geometry();

var periodTag = String(CONFIG.startYear) + '_' + String(CONFIG.endYear);

// Use MODIS native projection as the reference projection.
var referenceImage = ee.ImageCollection(CONFIG.datasetId)
  .filterDate(
    ee.Date.fromYMD(CONFIG.startYear, 1, 1),
    ee.Date.fromYMD(CONFIG.startYear, 2, 1)
  )
  .first()
  .select(CONFIG.nppBand);

var referenceProjection = referenceImage.projection();


/***************************************************************
 * 4. Annual NPP calculation
 ***************************************************************/

/**
 * Calculate annual NPP for a given year.
 *
 * @param {number|ee.Number} year Target year.
 * @return {ee.Image} Image with two bands: t and NPP.
 */
function calculateAnnualNpp(year) {
  year = ee.Number(year).toInt();

  var startDate = ee.Date.fromYMD(year, 1, 1);
  var endDate = startDate.advance(1, 'year');

  var annualNpp = ee.ImageCollection(CONFIG.datasetId)
    .filterDate(startDate, endDate)
    .select(CONFIG.nppBand)
    .sum()
    .multiply(CONFIG.scaleFactor)
    .rename('NPP')
    .clip(studyArea);

  var timeBand = ee.Image.constant(year)
    .toFloat()
    .rename('t')
    .updateMask(annualNpp.mask())
    .setDefaultProjection(annualNpp.projection());

  return timeBand
    .addBands(annualNpp)
    .set('year', year)
    .set('system:time_start', startDate.millis());
}


/***************************************************************
 * 5. Build annual NPP image collection
 ***************************************************************/

var years = ee.List.sequence(CONFIG.startYear, CONFIG.endYear);

var annualNppCollection = ee.ImageCollection.fromImages(
  years.map(calculateAnnualNpp)
);

print('Annual NPP collection:', annualNppCollection);


/***************************************************************
 * 6. Trend analysis
 ***************************************************************/

// Count valid annual observations for each pixel.
var sampleCount = annualNppCollection
  .select('NPP')
  .count()
  .rename('N');

// Valid pixels should have at least CONFIG.minSamples observations.
var validMask = sampleCount.gte(CONFIG.minSamples);

// Linear regression: NPP = a + S * year
var linearFit = annualNppCollection
  .select(['t', 'NPP'])
  .reduce(ee.Reducer.linearFit());

var slope = linearFit
  .select('scale')
  .rename('S')
  .updateMask(validMask)
  .clip(studyArea)
  .setDefaultProjection(referenceProjection);

// Pearson correlation and p-value
var correlation = annualNppCollection
  .select(['t', 'NPP'])
  .reduce(ee.Reducer.pearsonsCorrelation());

var pValue = correlation
  .select('p-value')
  .rename('P')
  .updateMask(validMask)
  .clip(studyArea)
  .setDefaultProjection(referenceProjection);


/***************************************************************
 * 7. Trend significance classification
 ***************************************************************/

var significant = pValue.lt(CONFIG.significanceLevel);
var positiveTrend = slope.gt(0);
var negativeTrend = slope.lt(0);

var trendClass = ee.Image(3)
  .toByte()
  .where(significant.and(positiveTrend), 1)
  .where(significant.and(negativeTrend), 2)
  .clip(studyArea)
  .rename('CLASS')
  .setDefaultProjection(referenceProjection);


/***************************************************************
 * 8. CV stability analysis
 ***************************************************************/

var meanNpp = annualNppCollection
  .select('NPP')
  .mean()
  .rename('MEAN');

var stdNpp = annualNppCollection
  .select('NPP')
  .reduce(ee.Reducer.stdDev())
  .rename('STD');

// CV = standard deviation / absolute mean
var cv = stdNpp
  .divide(meanNpp.abs())
  .rename('CV')
  .updateMask(validMask.and(meanNpp.neq(0)))
  .clip(studyArea)
  .setDefaultProjection(referenceProjection);

var cvClass = ee.Image(4)
  .toByte()
  .where(cv.lt(0.1), 1)
  .where(cv.gte(0.1).and(cv.lt(0.2)), 2)
  .where(cv.gte(0.2).and(cv.lt(0.3)), 3)
  .updateMask(cv.mask())
  .clip(studyArea)
  .rename('CV_CLASS')
  .setDefaultProjection(referenceProjection);


/***************************************************************
 * 9. Visualization
 ***************************************************************/

Map.centerObject(studyArea, CONFIG.mapZoom);

Map.addLayer(
  slope,
  {
    min: -20,
    max: 20,
    palette: ['#b2182b', '#f7f7f7', '#2166ac']
  },
  'NPP slope ' + periodTag
);

Map.addLayer(
  trendClass,
  {
    min: 1,
    max: 3,
    palette: ['#1a9850', '#d73027', '#cccccc']
  },
  'NPP trend class ' + periodTag
);

Map.addLayer(
  cvClass,
  {
    min: 1,
    max: 4,
    palette: ['#1a9850', '#91cf60', '#fee08b', '#d73027']
  },
  'NPP CV class ' + periodTag
);


/***************************************************************
 * 10. Export results
 ***************************************************************/

/**
 * Export image to Google Drive as GeoTIFF.
 *
 * @param {ee.Image} image Image to export.
 * @param {string} variableName Output variable name.
 */
function exportImageToDrive(image, variableName) {
  var lowerName = variableName.toLowerCase();

  Export.image.toDrive({
    image: image,
    description:
      CONFIG.outputPrefix.toUpperCase() +
      '_' +
      variableName +
      '_' +
      periodTag +
      '_WGS84',

    folder: CONFIG.exportFolder,

    fileNamePrefix:
      CONFIG.outputPrefix +
      '_' +
      lowerName +
      '_' +
      periodTag +
      '_wgs84',

    region: studyArea,
    crs: CONFIG.exportCrs,
    scale: CONFIG.exportScale,
    maxPixels: CONFIG.maxPixels,
    fileFormat: 'GeoTIFF'
  });
}

exportImageToDrive(slope, 'S');
exportImageToDrive(trendClass, 'CLASS');
exportImageToDrive(cvClass, 'CV_CLASS');


/***************************************************************
 * End of script
 ***************************************************************/
