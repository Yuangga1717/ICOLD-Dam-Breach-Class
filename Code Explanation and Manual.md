# Explanation Step 1 Input Data

folder = r"E:\2. Work\Bendungan Sampeyan Baru\1. Model\result final"

This is for the location where the folder path of the results will be shown. Include the extracted shapefile. put the flood depth and flood velocity tif file inside this folder

depth_file = os.path.join(folder, "2. depth overtopping p14 loc2.tif")


velocity_file = os.path.join(folder, "1. velocity overtopping p13 loc1.tif")
output_folder = os.getcwd()
os.makedirs(output_folder, exist_ok=True)

This is input data
