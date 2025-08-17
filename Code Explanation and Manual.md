  # Explanation Step 0 Input Data

    folder = r"E:\2. Work\Bendungan Sampeyan Baru\1. Model\result final"

This is for the location where the folder path of the results will be shown. Include the extracted shapefile. put the flood depth and flood velocity tif file inside this folder

Change this folder path to the folder containing the raster files for flood depth and flood velocity. 

    depth_file = os.path.join(folder, "2. depth overtopping p14 loc2.tif")

This is flood depth raster file, change this part 2. depth overtopping p14 loc2.tif as your flood depth raster file name

    velocity_file = os.path.join(folder, "1. velocity overtopping p13 loc1.tif")

This is flood velocity raster file, change this part 1. velocity overtopping p13 loc1.tif as your flood velocity raster file name

    output_folder = os.getcwd()
    os.makedirs(output_folder, exist_ok=True)

this is confirmation of folder location

# Explanation Step 1 Input Data
The rasterio.open() function is used to open raster files, such as GeoTIFFs. In this example, depth_file and velocity_file represent the file paths for the flood depth and velocity rasters, respectively. Using a 'with' statement ensures that the files are automatically closed after reading. 'src_d' and 'src_v' are 'rasterio' objects representing the opened rasters. The .read(1) function reads the first band of each raster, which is usually enough as most depth and velocity rasters only have one band. masked=True then converts any NoData values to masked values in a NumPy masked array, enabling the safe handling of missing or invalid data. Profile stores the raster metadata, including dimensions, resolution, coordinate reference system and data type. This is useful for creating new rasters, such as hazard maps, while preserving the original georeferencing.

    with rasterio.open(depth_file) as src_d, rasterio.open(velocity_file) as src_v:
        depth = src_d.read(1, masked=True)
        velocity = src_v.read(1, masked=True)
        profile = src_d.profile

# Explanation Step 2 Classifying hazard
The code loops through every cell in the raster; here, depth.shape[0] and depth.shape[1] represent the number of rows and columns, respectively. It skips cells where either depth or velocity is masked (NoData), using the statement if (depth.mask[i, j] or velocity.mask[i, j]) then continue. It then stores the depth and velocity values of the current cell in d and v, respectively.

The function np.interp() then interpolates the threshold velocity for the given depth. Here, v1 corresponds to the depth_line1 boundary (Low/Medium), while v2 corresponds to the depth_line2 boundary (Medium/High). This allows the hazard classification to adapt smoothly to the actual depth values rather than to discrete points. The actual velocity (v) is then compared with the interpolated thresholds: if v exceeds v1, the cell is classified as 'High hazard'; if v falls between v2 and v1, the cell is classified as 'Medium hazard'; otherwise, the cell is classified as 'Low hazard'.

    hazard_class = np.zeros(depth.shape, dtype=np.uint8)

    for i in range(depth.shape[0]):
        for j in range(depth.shape[1]):
            if depth.mask[i, j] or velocity.mask[i, j]:
                continue
            d = depth[i, j]
            v = velocity[i, j]

            v1 = np.interp(d, depth_line1, vel_line1)
            v2 = np.interp(d, depth_line2, vel_line2)

            if v > v1:
                hazard_class[i, j] = 3  # High
            elif v > v2:
                hazard_class[i, j] = 2  # Medium
            else:
                hazard_class[i, j] = 1  # Low

# Explanation Step 3 Plot hazard map
The code plt.figure(figsize=(8, 6)) creates a new figure measuring 8 by 6 inches. The line cmap = plt.cm.get_cmap("RdYlGn_r", 3) selects a colour map for the plot. Here, "RdYlGn_r" is a reversed red–yellow–green colour map, meaning that high values appear red and low values appear green. The number 3 then divides this into three categories corresponding to low, medium and high hazard. The line plt.imshow(hazard_class, cmap=cmap) displays the hazard_class array as an image, colouring each cell according to the colour map. plt.colorbar(ticks=[0, 1, 2, 3], label="Hazard class") adds a colour bar showing the mapping between colours and hazard classes. This colour bar has ticks for all classes (0 for background/NoData, 1 for Low, 2 for Medium and 3 for High) and a label describing it. plt.title("Hazard Classification Map") adds a title and plt.show() renders and displays the plot.

    plt.figure(figsize=(8,6))
    cmap = plt.cm.get_cmap("RdYlGn_r", 3)  # 3 classes
    plt.imshow(hazard_class, cmap=cmap)
    plt.colorbar(ticks=[0,1,2,3], label="Hazard class")
    plt.title("Hazard Classification Map")
    plt.show()

# Explanation Step 4 Export shapefiles per class
First, the code retrieves the georeferencing transform from the raster metadata (profile), which maps raster cell coordinates to real-world coordinates. This transform is necessary for converting raster cells to polygons. It then loops over the three hazard classes (1 = low, 2 = medium, 3 = high) and creates a Boolean mask identifying cells belonging to the current class. It then uses the function np.any(mask) to check if there are any cells in the class, skipping exporting if there are none and printing a message indicating which class is being exported. The shapes() function from rasterio.features generates vector polygons from raster regions of the same value. Hazard_class.astype(np.int16) ensures the format is integer, mask=mask limits conversion to the current class and transform=transform ensures proper georeferencing. The generated shapes are then converted into a GeoDataFrame using Geopandas. This stores each polygon’s geometry and hazard class, filters to include only polygons where val is equal to the current hazard class (cls), and sets the coordinate reference system to match the original raster (crs=src_d.crs). Finally, the code constructs the output shapefile path for the current class, saves the GeoDataFrame as a shapefile using gdf.to_file() and prints a message to confirm that the shapefile has been saved.

    transform = profile["transform"]

    for cls in [1, 2, 3]:
        mask = hazard_class == cls
        if np.any(mask):
            print(f"  -> Exporting class {cls}...")
            shapes_gen = shapes(hazard_class.astype(np.int16), mask=mask, transform=transform)

            gdf = gpd.GeoDataFrame(
                [{"geometry": shape(geom), "hazard": cls} for geom, val in shapes_gen if val == cls],
                crs=src_d.crs
            )
    
            out_shp = os.path.join(output_folder, f"hazard_class_{cls}.shp")
            gdf.to_file(out_shp)
