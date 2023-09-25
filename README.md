# preprocessing_planet-imagery
# Install libraries
import rasterio
from os import walk
from rasterio.windows import Window
from rasterio.merge import merge
from rasterio.mask import mask
import gc 
from xml.dom import minidom
import geopandas as gpd
import numpy as np


n_images = 2
Project_folder = 'D:/Planet_2023/'
AnalyticMS_folder = 'D:/Planet_2023/Analytics_MS/'
Metadata_folder = 'D:/Planet_2023/Metadata/'
aoi_folder = 'D:/Planet_2023/AOI/'
Results_folder = 'D:/Planet_2023/Results/'
# Collect image filenames 
from os import walk
AnalyticMS_files = []
for (dirpath, dirnames, filenames) in walk(AnalyticMS_folder):
    AnalyticMS_files.extend(filenames)
    break

# Add image file path
AnalyticMS_path = []
for i in AnalyticMS_files:
    AnalyticMS_path.append(AnalyticMS_folder + i)
    print(AnalyticMS_folder + i) # Verify correct images
# Reading image metadata
Analytic_meta_files = []
for (dirpath, dirnames, filenames) in walk(Metadata_folder):
    Analytic_meta_files.extend(filenames)
    break

# Add image metadata file path
Analytic_meta_path = []
for i in Analytic_meta_files:
    Analytic_meta_path.append(Metadata_folder + i)
    print(Metadata_folder + i) # Verify correct metadata
# Loading images into list using rasterio
img_list = []
for image in AnalyticMS_path:
    img_list.append(rasterio.open(image))
print('Length of img_list: {} '.format(len(img_list)))
# Inspecting image metadata
print("Image: dtype | crs | band count")
for image in img_list:
    print(image.meta['dtype'], image.meta['crs'], image.meta['count'])
# Read in color interpretations of each band in img1 - assume same values for rest of the images
colors = [img_list[0].colorinterp[band] for band in range(img_list[0].count)]

# taking a look at img1's band types:
for color in colors:
    print(color.name)
# Radiance values are loaded into lists
band_blue_radiance = [0]*n_images
band_green_radiance = [0]*n_images
band_red_radiance = [0]*n_images
band_nir_radiance = [0]*n_images

# Using rasterio to read radiance values for the images
for i in range(n_images):
    with rasterio.open(AnalyticMS_path[i]) as src:
        band_blue_radiance[i] = src.read(1)
    with rasterio.open(AnalyticMS_path[i]) as src:
        band_green_radiance[i] = src.read(2)
    with rasterio.open(AnalyticMS_path[i]) as src:
        band_red_radiance[i] = src.read(3)
    with rasterio.open(AnalyticMS_path[i]) as src:
        band_nir_radiance[i] = src.read(4)
# Gathering the coefficients for the 7 images
coeffs_list = [] 
for image in Analytic_meta_path:
    xmldoc = minidom.parse(image)
    nodes = xmldoc.getElementsByTagName("ps:bandSpecificMetadata")
    
    # XML parser refers to bands by numbers 1-4
    coeffs = {} # Coefficients for each image/scene
    for node in nodes:
        band_nr = node.getElementsByTagName("ps:bandNumber")[0].firstChild.data
        if band_nr in ['1', '2', '3', '4']:
            i = int(band_nr)
            value = node.getElementsByTagName("ps:reflectanceCoefficient")[0].firstChild.data
            coeffs[i] = float(value)
    coeffs_list.append(coeffs)

for coffee in coeffs_list:
    print (coffee) # Verify data/coffee
# Radiance values are loaded into lists
band_blue_reflectance = [];   blue_ref_scaled = [0]*n_images
band_green_reflectance = [];  green_ref_scaled = [0]*n_images
band_red_reflectance = [];    red_ref_scaled = [0]*n_images
band_nir_reflectance = [];    nir_ref_scaled = [0]*n_images

# Calculating reflectance from radiance and conversion coefficients
for i in range(n_images):
        band_blue_reflectance.append(band_blue_radiance[i] * coeffs_list[i][1])
        band_green_reflectance.append(band_green_radiance[i] * coeffs_list[i][2])
        band_red_reflectance.append(band_red_radiance[i] * coeffs_list[i][3])
        band_nir_reflectance.append(band_nir_radiance[i] * coeffs_list[i][4])
        
# Check the results by looking at the min-max range of the radiance (before) vs reflectance (after)
# for the red band from the first image, excluding NaN values
red_rad_min = np.nanmin(band_red_radiance[0]);    red_rad_max = np.nanmax(band_red_radiance[0])
red_ref_min = np.nanmin(band_red_reflectance[0]); red_ref_max = np.nanmax(band_red_reflectance[0])
print("Red band radiance goes from: {} to {}".format(red_rad_min, red_rad_max))
print("Red band reflectance goes from: {} to {}".format(red_ref_min, red_ref_max))
# Function to export scaled images - (alternative option)
def scaled_export(images):
    for i in range(len(images)):
        # Get the metadata of original GeoTIFF:
        meta = images[i].meta
        
        # Set the source metadata as kwargs we'll use to write the new data:
        # update the 'dtype' value to uint16:
        kwargs = meta
        kwargs.update(dtype='uint16')
        
        # As noted above, scale reflectance value by a factor of 10k:
        scale = 10000
        blue_ref_scaled[i] = scale * band_blue_reflectance[i]
        green_ref_scaled[i] = scale * band_green_reflectance[i]
        red_ref_scaled[i] = scale * band_red_reflectance[i]
        nir_ref_scaled[i] = scale * band_nir_reflectance[i]
        
        # Compute new min & max values for the scaled red band, just for comparison
        red_min_scaled = np.amin(red_ref_scaled[0])
        red_max_scaled = np.amax(red_ref_scaled[0])
        
        # Convert to type 'uint16'
        from rasterio import uint16
        blue_scaled = blue_ref_scaled[i].astype(uint16)
        green_scaled = green_ref_scaled[i].astype(uint16)
        red_scaled = red_ref_scaled[i].astype(uint16)
        nir_scaled = nir_ref_scaled[i].astype(uint16)
        
        # New name for exported image
        img_name_ref_scaled = (Results_folder + AnalyticMS_files[i]).replace('.tif','_Ref.tif')
        
        # Write band calculations to a new unscaled GeoTIFF file with same band order (BGRN)
        with rasterio.open(img_name_ref_scaled, 'w', **kwargs) as dst:
                dst.write_band(1, blue_scaled)
                dst.write_band(2, green_scaled)
                dst.write_band(3, red_scaled)
                dst.write_band(4, nir_scaled)
         # Comparing min & max values before/after scaling, with the first image red band
        if i == 0:
            print("With scale factor: {}".format(scale))
            print("Before scaling: Red band reflectance from: {} to {}"\
                  .format(red_ref_min, red_ref_max))
            print("After scaling: Red band reflectance from: {} to {}"\
                  .format(red_min_scaled,red_max_scaled))
        if i == (len(images)-1):
            print("Success, {} scaled images(dtype={}) have been exported to your Results folder: {}"\
                  .format(n_images, kwargs['dtype'], Results_folder ))
# Export scaled images:
scaled_export(img_list)
