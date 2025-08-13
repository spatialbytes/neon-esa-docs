# Spectral Unmixing with NEON Hyperspectral Data

!!! info ""

    === "📋 Tutorial Details"
        - **Duration:** 10 minutes
        - **Level:** Intermediate
        - **Authors:** Samapriya Roy

    === "🏷️ Topics & Data"
        - **Topics:** lidar, hyperspectral, camera, remote-sensing
        - **Data Products:** DP3.30006.001, DP3.30006.002, DP3.30010.001, DP3.30015.001, DP3.30024.001

## Objectives

After completing this activity, you will be able to:

- Load and filter NEON hyperspectral imagery to remove problematic spectral bands
- Extract pure endmember spectra from representative land cover areas using reducers
- Apply constrained spectral unmixing with non-negativity and sum-to-one constraints
- Visualize fractional abundance maps for different land cover types

## Requirements

- Basic understanding of Google Earth Engine and JavaScript
- Familiarity with hyperspectral remote sensing concepts
- Knowledge of spectral signatures and endmember analysis
- Understanding of NEON AOP data products

## Load NEON Hyperspectral Data

We'll start by loading the NEON Bidirectional Reflectance Hyperspectral Image Collection, which contains hundreds of spectral bands covering visible to near-infrared wavelengths:

```javascript
// Load NEON Bidirectional Reflectance Hyperspectral Image Collection
var neon_bidirectional = ee.ImageCollection("projects/neon-prod-earthengine/assets/HSI_REFL/002");

// Explore available NEON sites in the collection
print('Available NEON sites:', neon_bidirectional.aggregate_histogram('NEON_SITE'));
```

## Remove Problematic Spectral Bands

Hyperspectral data often contains bands affected by atmospheric absorption or sensor noise. We need to identify and remove these problematic bands before analysis:

```javascript
// Define problematic spectral bands to exclude from analysis
// These bands are typically excluded due to:
// - Atmospheric absorption features (water vapor, oxygen)
// - Sensor noise or calibration issues
// - Edge effects at sensor boundaries
var bands_to_remove = [
  'B195','B196','B197','B198','B199','B200','B201','B202','B203','B204','B205', // Water vapor ~940nm
  'B287','B288','B289','B290','B291','B292','B293','B294','B295','B296','B297','B298', // Water vapor ~1130-1160nm
  'B299','B300','B301','B302','B303','B304','B305','B306','B307','B308','B309','B310', // Continued water vapor
  'B416','B417','B418','B419','B420','B421','B422','B423','B424','B425' // Water vapor ~1400nm
];

var BAD_BAND_NAMES = ee.List(bands_to_remove);
```

!!! note "Why Remove These Bands?"
    Water vapor absorption windows around 940nm, 1130-1160nm, and 1400nm cause atmospheric interference that makes reflectance values unreliable. Removing these bands improves unmixing accuracy.

## Create Band Filtering Functions

Next, we'll create utility functions to systematically remove the problematic bands from our imagery:

```javascript
// Filter an image to retain only valid B-bands (spectral bands)
// Excludes problematic bands and non-spectral bands (metadata, etc.)
function selectValidBands(image) {
  var allBands = image.bandNames(); // Get all band names
  var bBands = allBands.filter(ee.Filter.stringStartsWith('item', 'B')); // Keep only B-bands
  var validBands = bBands.filter(ee.Filter.inList('item', BAD_BAND_NAMES).not()); // Exclude bad bands
  return image.select(validBands);
}
```

## Select and Prepare Site Data

We'll focus on SERC (Smithsonian Environmental Research Center), which represents a temperate deciduous forest ecosystem:

```javascript
// Select and prepare data from SERC
var serc_site = neon_bidirectional.filter(ee.Filter.eq('NEON_SITE','SERC')).first();
var serc_filtered = selectValidBands(serc_site);

// Display band filtering results for quality control
print('Original bands:', serc_site.bandNames().size());
print('Filtered bands:', serc_filtered.bandNames().size());
print('Number of excluded bands:', BAD_BAND_NAMES.size());

var image = serc_filtered;
Map.centerObject(image);
```

!!! tip "Quality Control Check"
    Always verify the number of bands before and after filtering to ensure the process worked correctly. NEON hyperspectral data typically has 426 bands, and you should retain most of them after removing problematic bands.

## Extract Pure Endmember Spectra Using Reducers

Endmembers represent the "pure" spectral signatures of different land cover types. We use the `reduceRegion` function to extract mean spectral signatures from representative areas:

```javascript
// Define training data collections (these should be drawn as geometries)
var ag = /* color: #ffc82d */ ee.FeatureCollection([]);      // Agricultural areas
var water = /* color: #2564ff */ ee.FeatureCollection([]);   // Water bodies
var forest = /* color: #2ac219 */ ee.FeatureCollection([]);  // Forest cover

// Extract water endmember - represents open water spectral signature
// Typically shows low reflectance across all bands
var water_endmember = image.reduceRegion({
  reducer: ee.Reducer.mean(),
  geometry: water,
  scale: 1, // Native NEON resolution (1m)
  maxPixels: 1e9,
  bestEffort: true
}).values();

// Extract forest canopy endmember - represents dense vegetation
// Shows characteristic vegetation features: low red, high NIR reflectance
var canopy_endmember = image.reduceRegion({
  reducer: ee.Reducer.mean(),
  geometry: forest,
  scale: 1,
  maxPixels: 1e9,
  bestEffort: true
}).values();

// Extract agricultural endmember - represents crop/agricultural areas
var ag_endmember = image.reduceRegion({
  reducer: ee.Reducer.mean(),
  geometry: ag,
  scale: 1,
  maxPixels: 1e9,
  bestEffort: true
}).values();

print('Water endmember values:', water_endmember);
```

!!! note "Endmember Selection Strategy"
    Choose the purest possible pixels for each land cover type. Water areas should be deep and clear, forest areas should be dense canopy, and agricultural areas should be uniform crop fields.

## Apply Constrained Spectral Unmixing

Now we'll perform the actual spectral unmixing using the extracted endmembers. The constraints ensure physically meaningful results:

```javascript
// Organize endmembers for spectral unmixing
// Order matters: results will correspond to this sequence
var endmembers = [ag_endmember, canopy_endmember, water_endmember];
var endmember_names = ['agriculture', 'forestry', 'water'];

// Perform constrained spectral unmixing
// Parameters:
// - endmembers: Array of pure spectral signatures
// - sumToOne: true = force fractions to sum to 1.0 (Sum-to-One Constraint)
// - nonNegative: true = constrain fractions to be non-negative (Non-negativity Constraint)
var unmixed = image.unmix(endmembers, true, true).rename(endmember_names);

// Display unmixed results on map
Map.addLayer(unmixed, {}, 'Unmixed Fractional Abundances');
```

## Understanding the Constraints

The constrained spectral unmixing applies two important physical constraints. You can read [the paper here](https://ieeexplore.ieee.org/abstract/document/974727)

### Non-negativity Constraint (ANC)
```
a_i ≥ 0 for all i
```
Ensures that abundance fractions are physically meaningful (no negative abundances). You cannot have "negative vegetation" in a pixel.

### Sum-to-One Constraint (ASC)
```
Σ a_i = 1
```
Ensures that abundance fractions sum to 100%, representing complete pixel coverage. This assumes that your endmembers represent all materials present in the scene.

!!! tip "Interpreting Results"
    Each band in the unmixed image represents fractional abundance (0-1) of the corresponding land cover type. Values closer to 1 indicate higher abundance of that material in the pixel.

## Visualization and Analysis

The resulting unmixed image contains three bands representing the fractional abundance of each endmember:

- **Agriculture band**: Shows areas with high crop/agricultural content
- **Forestry band**: Shows areas with high forest canopy content
- **Water band**: Shows areas with high water content

```javascript
// Create individual abundance maps for better visualization
var ag_abundance = unmixed.select('agriculture');
var forest_abundance = unmixed.select('forestry');
var water_abundance = unmixed.select('water');

// Add individual layers with appropriate color schemes
Map.addLayer(ag_abundance, {min: 0, max: 1, palette: ['white', 'yellow']}, 'Agriculture Abundance');
Map.addLayer(forest_abundance, {min: 0, max: 1, palette: ['white', 'green']}, 'Forest Abundance');
Map.addLayer(water_abundance, {min: 0, max: 1, palette: ['white', 'blue']}, 'Water Abundance');
```

## Summary

In this lesson you learned how to perform constrained spectral unmixing with NEON hyperspectral data. Key steps included filtering out problematic spectral bands affected by atmospheric absorption, using reducers to extract pure endmember spectra from representative land cover areas, and applying constrained unmixing with non-negativity and sum-to-one constraints. The resulting fractional abundance maps provide quantitative estimates of land cover composition at the pixel level.
