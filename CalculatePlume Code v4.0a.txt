# =========================================================================================
# Plume Tool; Waste Estimation Support Tool (WEST) US EPA
# Created by: Timothy Boe, Paul Lemieux
# Last Modified: 05.11.15; WEST v3
# Description: 1) Loads, merges (if necessary), and massages plume shapefile; 2) Establishes the area and distribution of the plume within the census tracts; 3) Generates a KML files of the census tracts; 4) Capture imagery clipped to the plume
# Input:  Shapefile
# Output: Census.txt, Output.csv, Plumearea.csv, doc.kml, Zone1.bmp, Zone2.bmp, Zone3.bmp
# =========================================================================================

# Import modules
import arcpy, csv, sys, os, zipfile, fileinput, contextlib, shutil
from arcpy.sa import *

#!!!Scenario Management!!!

#WEST ROOT = west base path
#Working Folder = WEST GIS folder
#Working Folder Project = Scenario folder within working folder

#Establish project name.
Project_Name = arcpy.GetParameterAsText(0)

#Establish base path
WEST_Root = os.path.abspath(os.path.dirname(__file__))
arcpy.AddMessage("WEST Root Directory: " + WEST_Root)

#Establish working folder. If it doesn't exist, create
Working_Folder = os.path.join(WEST_Root, "GIS_ImportFiles")

if not os.path.exists(Working_Folder):
    os.makedirs(Working_Folder)

#Establish project folder. Do not delete duplicate. Throw error if duplicate name encountered
Working_Folder_Project = os.path.join(Working_Folder, Project_Name)

if os.path.exists(Working_Folder_Project):
    try:
        shutil.rmtree(Working_Folder_Project)
        os.makedirs(Working_Folder_Project)
        arcpy.AddWarning("GIS Scenario folder not found. Creating folder...")
    except:
        arcpy.AddError("Duplicate scenario folder found. Attempting to delete...")
        arcpy.AddError("ERROR: The scenario " + Project_Name + " already exists and is currently in use. To prevent the loss of critical files, WEST is unable to finish. To continue, either change the scenario name or close ArcGIS and delete the scenario folder: " + Project_Name + " from the WEST GIS Scenarios folder located here: " + Working_Folder)
        sys.exit("Error")
else:
    try:
        os.makedirs(Working_Folder_Project)
        arcpy.AddMessage("Establishing work space")
    except:
        arcpy.AddError("ERROR: Unable to create scenario folder. This may be due to limited read/write rights.")
        sys.exit("Error")

arcpy.AddMessage("Done!")
   
# Determine scenario name
Folder = Working_Folder_Project
FileName = os.path.basename(Working_Folder_Project)
KML_out = os.path.basename(Working_Folder_Project)
FileName = FileName.replace(" ", "_")

# Plume file name lookup
Plume_File = "Plume_" + FileName

# Global variables
arcpy.AddMessage("Loading global variables")
x = arcpy.GetParameterAsText(1)
BarePlume = arcpy.GetParameterAsText(2)
Zone_2 = arcpy.GetParameterAsText(3)
Zone_1 = arcpy.GetParameterAsText(4)
Imagery_Type = arcpy.GetParameterAsText(5)
Imagery_Location = arcpy.GetParameterAsText(6)
Study_Region_Tract_Proj = os.path.join(Folder, "Study_Region_Tract_Proj.shp")
Study_Region_Tract_New = os.path.join(Folder, "Study_Region_Tract_New.shp") #Fix this later
arcpy.AddMessage("Done!")

# Map and file arguments
arcpy.AddMessage("Loading map and file arguments")
mxd=arcpy.mapping.MapDocument('CURRENT')
df = arcpy.mapping.ListDataFrames(mxd,"*")[0]
ColorCode = os.path.join(WEST_Root, "ColorCode.txt")
KMLFile = os.path.join(Folder, "doc.kml")
Plume_Sort = Folder + "\Plume_" + FileName + ".shp"
Study_Region_Tract = "Study Region Tract"
arcpy.AddMessage("Done!")

#######################
# Function: Census Tract Pull - Updated on 6.5.18
# This block of code pulls in a census tract feature layer that will be clipped specified to the plume polygon
#######################

arcpy.AddMessage("Loading Census Tracts. This may take a few minutes...")
Study_Region_Tract = arcpy.mapping.Layer(os.path.join(WEST_Root, "Tract.lyr"))
arcpy.mapping.AddLayer(df, Study_Region_Tract,"TOP")

# Set name of output fc and select buildings by location
arcpy.SelectLayerByLocation_management(Study_Region_Tract, "INTERSECT", BarePlume, "", "NEW_SELECTION","NOT_INVERT")
arcpy.CopyFeatures_management(Study_Region_Tract, Study_Region_Tract_New)
arcpy.AddMessage("Done!")

#!!!End Census Tract Pull!!!

# Establish workspace 6.7.18
arcpy.AddMessage("Establishing workspace")
West_Workspace = Folder + "\WEST1.gdb"
arcpy.env.workspace = West_Workspace
arcpy.AddMessage(arcpy.env.workspace)
arcpy.AddMessage("Done!")

# Enable overwrite
arcpy.env.overwriteOutput = True

# Set CenShape_shp parameter to folder and add CenPlume filename and .shp extension
arcpy.AddMessage("Loading local variables")
CenShape_shp = Folder + "\CenPlume.shp"
CenShape_Sort = Folder + "\CenPlume_Sort.shp"
arcpy.AddMessage("Done!")

# Local variables:
arcpy.AddMessage("Loading local variables")
BarePlume__3_ = BarePlume
BarePlume__4_ = BarePlume__3_
BarePlume__2_ = BarePlume__4_
Plume_Sort__2_ = Plume_Sort
arcpy.AddMessage("Done!")

#######################
# Function: Build Plume - Updated on 6.6.18
# This block of code modifies the plume file to ensure compatibility with WEST
#######################

# Check boolean for true; if true, merge plumes, otherwise pass to next function
if x == "Multi Layer Plume (load plumes in zones 3, 2, 1,)":
    arcpy.AddMessage("Merging plumes")
    Plume_fixed = Folder + "\Plume_fixed.shp"
# Merge plumes in the following order: zone3>zone2>zone1
    arcpy.Merge_management([BarePlume, Zone_2, Zone_1], Plume_fixed, "")
    arcpy.AddMessage("Done!")
# Agree local variables
    arcpy.AddMessage("Checking local variables")
    BarePlume__3_ = Plume_fixed
    BarePlume__4_ = BarePlume__3_
    BarePlume__2_ = BarePlume__4_
    Plume_Sort__2_ = Plume_Sort
    arcpy.AddMessage("Done!")
else:
    Plume_fixed = BarePlume

# Process: Add field "Plumearea" as double; field is nullable
arcpy.AddMessage("Adding field Plumearea")
arcpy.AddField_management(Plume_fixed, "Plumearea", "DOUBLE", "", "", "", "", "NULLABLE", "NON_REQUIRED", "")
arcpy.AddMessage("Done!")

# Process: Add field "LINE_ID" as text; field is nullable
arcpy.AddMessage("Adding field LINE_ID")
arcpy.AddField_management(BarePlume__3_, "LINE_ID", "TEXT", "", "", "", "", "NULLABLE", "NON_REQUIRED", "")
arcpy.AddMessage("Done!")

# Process: Calculate plume area in m^2 using "!shape.area@squaremeters!"
arcpy.AddMessage("Calculating plume area")
arcpy.CalculateField_management(BarePlume__4_, "Plumearea", "!shape.area@squaremeters!", "PYTHON", "")
arcpy.AddMessage("Done!")

# Process: Sort records in attribute table by "Plumearea" in descending order
arcpy.AddMessage("Sorting plume by area")
arcpy.Sort_management(BarePlume__2_, Plume_Sort, "Plumearea DESCENDING", "UR")
arcpy.AddMessage("Done!")

# Process: Starting at the top line (<area), iterate through table, assigning each record a plume ID by a decrement of 1
arcpy.AddMessage("Generating plume levels")
arcpy.CalculateField_management(Plume_Sort, "LINE_ID", "autoIncrement()", "PYTHON", "rec = 0\\ndef autoIncrement(): \\n    global rec\\n    pStart = 3  \\n    pInterval = 1 \\n    if (rec == 0):  \\n        rec = pStart  \\n    else:  \\n        rec -= pInterval\\n    return \"Level00\" + str(rec)")
arcpy.AddMessage("Done!")

#######################
# Function: Explode Plume - Updated on 6.5.18
# This block of code explodes and rebuilds plume files to prevent overlap within the polygon.
#######################

# Create feature layer for next series of functions to operate correctly
arcpy.MakeFeatureLayer_management(Plume_Sort, 'Plume_Sort_New')

# Select Zone or Level 1
arcpy.AddMessage("Clipping zone 1 from plume")
val = "Level001"
field = arcpy.AddFieldDelimiters('Plume_Sort_New', "LINE_ID")
selection = "{field} = '{val}'".format(field=field, val=val)
arcpy.SelectLayerByAttribute_management('Plume_Sort_New', "NEW_SELECTION", selection)

# Create new feature consisting of Zone or Level 1
arcpy.CopyFeatures_management('Plume_Sort_New', 'Zone1')
arcpy.AddMessage("Done!")

# Select Zone or Level 2
arcpy.AddMessage("Clipping zone 2 from plume")
val = "Level002"
field = arcpy.AddFieldDelimiters('Plume_Sort_New', "LINE_ID")
selection = "{field} = '{val}'".format(field=field, val=val)
arcpy.SelectLayerByAttribute_management('Plume_Sort_New', "NEW_SELECTION", selection)
# Create new feature consisting of Zone or Level 2
arcpy.CopyFeatures_management('Plume_Sort_New', 'Zone2')
arcpy.AddMessage("Done!")

# Select Zone or Level 3
arcpy.AddMessage("Clipping zone 3 from plume")
val = "Level003"
field = arcpy.AddFieldDelimiters('Plume_Sort_New', "LINE_ID")
selection = "{field} = '{val}'".format(field=field, val=val)
arcpy.SelectLayerByAttribute_management('Plume_Sort_New', "NEW_SELECTION", selection)

# Create new feature consisting of Zone or Level 3
arcpy.CopyFeatures_management('Plume_Sort_New', 'Zone3')
arcpy.AddMessage("Done!")

# Clear selection
arcpy.SelectLayerByAttribute_management('Plume_Sort_New', "CLEAR_SELECTION")

##FIX##
# Merge plumes using Update function. Start with largest zone (Zone 3). This function rebuilds the plume while keeping the contours intact. Depending on the interection of the polygons, unnecessary polygons/filed can be created. The next process seeks to resolve this
arcpy.Update_analysis('Zone3', 'Zone2', 'Z2_Z3', 
                      "BORDERS",)

arcpy.Update_analysis('Z2_Z3', 'Zone1', 'plume_clean', 
                      "BORDERS",)

# Make feature layer for compatibility with "select layer" command
arcpy.MakeFeatureLayer_management("plume_clean", "plume_clean")

arcpy.AddMessage("Applying finishing touches to plume")
# Select fields that contain an FID[Layer] of -1 which indicates a gap within the polygon 
#val = "-1"
#field = arcpy.AddFieldDelimiters('plume_clean', "FID_Zone3")
#selection = "{field} = {val}".format(field=field, val=val)
#arcpy.SelectLayerByAttribute_management('plume_clean', "NEW_SELECTION", selection)

# Delete the selected rows
#arcpy.DeleteRows_management('plume_clean')
#arcpy.AddMessage("Done!")

# Clear the selection
arcpy.SelectLayerByAttribute_management('plume_clean', "CLEAR_SELECTION")

#######################
# Function: Rebuild Exploded Plume - Updated on 6.5.18
# This block of code rebuilds the plume following the union. The union causes mislabeling of features -- this function repairs those issues
#######################

# Process: Calculate plume area in m^2 using "!shape.area@squaremeters!"
arcpy.AddMessage("Calculating plume area")
arcpy.CalculateField_management('plume_clean', "Plumearea", "!shape.area@squaremeters!", "PYTHON", "")
arcpy.AddMessage("Done!")

# Process: Sort records in attribute table by "Plumearea" in descending order
arcpy.AddMessage("Sorting plume by area")
arcpy.Sort_management('plume_clean', 'plume_clean_sort', "Plumearea DESCENDING", "UR")
arcpy.AddMessage("Done!")

# Process: Starting at the top line (<area), iterate through table, assigning each record a plume ID by a decrement of 1
arcpy.AddMessage("Generating plume levels")
arcpy.CalculateField_management('plume_clean_sort', "LINE_ID", "autoIncrement()", "PYTHON", "rec = 0\\ndef autoIncrement(): \\n    global rec\\n    pStart = 3  \\n    pInterval = 1 \\n    if (rec == 0):  \\n        rec = pStart  \\n    else:  \\n        rec -= pInterval\\n    return \"Level00\" + str(rec)")
arcpy.AddMessage("Done!")

#Create a new layer
arcpy.AddMessage("Generating new layer")
newplumelayer = arcpy.mapping.Layer('plume_clean_sort')
arcpy.AddMessage("Done!")

# Delete comment instruction below for troubleshooting the above code
#arcpy.CopyFeatures_management('plume_clean_sort', os.path.join(WEST_Root, "Clipped_Plume_Troubleshoot.shp"))

# Add the layer to the map at the bottom of the TOC in data frame 0
arcpy.AddMessage("Adding ayer to Map Document")
arcpy.mapping.AddLayer(df, newplumelayer,"BOTTOM")
arcpy.AddMessage("Done!")

# Check for Plume_fixed.shp existence
if x == "Multi Layer Plume (load plumes in zones 3, 2, 1,)":
    # Delete temporary plumes
    arcpy.AddMessage("Obliterating temporary plumes")
    #arcpy.Delete_management(Plume_fixed)
    arcpy.AddMessage("Done!")
else:
    pass

# Define Plume feature; this is used to convert feature generated above so that it can be used as a feature layer is in following processes
#6.5.18
Plume = "Plume"
arcpy.MakeFeatureLayer_management('plume_clean_sort', Plume)

#######################
# Function: Calculate Plume - Updated on ---
# This block of code calculate surface area and intersect plume with census tracts
#######################

# Local variables
Plume__2_ = Plume
Plume__3_ = Plume__2_
Plume_Intersect__2_ = CenShape_shp
Plume_Intersect__3_ = Plume_Intersect__2_
CenShape_shp__2_ = Plume_Intersect__3_
CenShape_shp__3_ = CenShape_shp__2_

# Project census tract shapefile; this is necessary in order to determine the area of the census tracts
arcpy.AddMessage("Projecting census tracts")
# Project to USA_Contiguous_Albers_Equal_Area_Conic

out_coordinate_system = arcpy.SpatialReference(102003)
arcpy.Project_management(Study_Region_Tract_New, Study_Region_Tract_Proj, out_coordinate_system)
arcpy.AddMessage("Done!")

# Add Field "Census Tract Area" as double; field is nullable
arcpy.AddMessage("Adding field Plume Area")
arcpy.AddField_management(Study_Region_Tract_Proj, "SHAPE_Area", "DOUBLE", "", "", "", "", "NULLABLE", "NON_REQUIRED", "")
arcpy.AddMessage("Done!")

# Calculate area of census tracts
arcpy.AddMessage("Calculating census tract area")
arcpy.CalculateField_management(Study_Region_Tract_Proj, "SHAPE_Area", "!shape.area@squaremeters!", "PYTHON", "")
arcpy.AddMessage("Done!")

# Add Field "Plume Area" as double; field is nullable
arcpy.AddMessage("Adding field Plume Area")
arcpy.AddField_management(Plume, "Plumearea", "DOUBLE", "", "", "", "", "NULLABLE", "NON_REQUIRED", "")
arcpy.AddMessage("Done!")

# Calculate Field "Plume Area" in m^2 using "!shape.area@squaremeters!"
arcpy.AddMessage("Calculating plume area")
arcpy.CalculateField_management(Plume__2_, "Plumearea", "!shape.area@squaremeters!", "PYTHON", "")
arcpy.AddMessage("Done!")

# Intersect "Study Region Tract" and "Plume"
arcpy.AddMessage("Intersecting Study Region and Plume")
arcpy.Intersect_analysis([Study_Region_Tract_Proj, Plume__2_], CenShape_shp, "ALL", "", "INPUT")
arcpy.AddMessage("Done!")

# Add Field "Area_m2b" to store m^2 after intersect; as double; field is nullable
arcpy.AddMessage("Adding field Area_m2b")
arcpy.AddField_management(CenShape_shp, "Area_m2b", "DOUBLE", "", "", "", "", "NULLABLE", "NON_REQUIRED", "")
arcpy.AddMessage("Done!")

# Add Field "CenTract" to store census tract percentages; as double; field is nullable
arcpy.AddMessage("Adding field Centract")
arcpy.AddField_management(Plume_Intersect__2_, "CenTract", "DOUBLE", "", "", "", "", "NULLABLE", "NON_REQUIRED", "")
arcpy.AddMessage("Done!")

# Calculate Field "Area_m2b" in m^2 using "!shape.area@squaremeters!"
arcpy.AddMessage("Updating plume area")
arcpy.CalculateField_management(Plume_Intersect__3_, "Area_m2b", "!shape.area@squaremeters!", "PYTHON", "")
arcpy.AddMessage("Done!")

# Calculate Field Centract; divide "Area_m2b" by "Shape_Area" and * 100; returns percentage of tract encompassed by plume
arcpy.AddMessage("Calculating percentages")
arcpy.CalculateField_management(CenShape_shp__2_, "CenTract", "!Area_m2b! / !SHAPE_Area! *100", "PYTHON_9.3", "")
arcpy.AddMessage("Done!")

#######################
# Function: Create KML - Updated on ---
# This block of code creates a KML file for use by WEST
#######################

# Convert shapefile to feature layer for exporting KML
arcpy.AddMessage("Converting plume file to feature layer")
arcpy.MakeFeatureLayer_management(CenShape_shp__2_, FileName)
arcpy.AddMessage("Done!")

# Add layer to TOC so parts can be labeled
newlayer = arcpy.mapping.Layer(FileName)
arcpy.mapping.AddLayer(df, newlayer,"BOTTOM")

# Label plume per tract ID. This is necessary in order for tract IDs to retain name when converting to KML
arcpy.AddMessage("Assigning census tract IDs to KMZ")
layer = newlayer 
if layer.supports("LABELCLASSES"):
    for lblclass in layer.labelClasses:
        lblclass.showClassLabels = True
lblclass.expression = "[tract]"
layer.showLabels = True
arcpy.RefreshActiveView()
arcpy.AddMessage("Done!")

# KML Outbound file
KML_ouput = os.path.join(Folder, FileName) + ".kmz"

# Create KMZ
arcpy.AddMessage("Saving KMZ file")
arcpy.LayerToKML_conversion(FileName, KML_ouput, "10000", "false", "DEFAULT", "1024", "96")
arcpy.AddMessage("Done!")

source = os.path.join(Folder, FileName) + ".kmz"
target = Folder

# Extract KML file from KMZ as Zip
arcpy.AddMessage("Converting KMZ to KML")
with contextlib.closing(zipfile.ZipFile(KML_ouput, "r")) as z:
    z.extractall(Folder)
arcpy.AddMessage("Done!")

# Open color code file and insert text into KML. These data will establish color codes & imagery
arcpy.AddMessage("Rebuilding KML file")
with open (ColorCode, "r") as myfile:
    data=myfile.read()   
process = False
for line in fileinput.input(KMLFile, inplace=1):
  if line.startswith('  <name>' + FileName + '</name>'):
    process = True
  else:
    if process:
      print data
    process = False
  print line,
arcpy.AddMessage("Done!")

# Ensure that kml file does not already exist before renaming
try:
    os.remove(os.path.join(Folder, FileName + ".kml"))
    arcpy.AddMessage("Warning: KML file with the same name already exists. Banishing KML file...")
    arcpy.AddMessage("Done!")
except OSError:
    pass

# Rename KML file to reflect scenario name
# Made an update here 5.11.15
os.rename(os.path.join(Folder,"doc.kml"),os.path.join(Folder, KML_out + ".kml"))

#######################
# Function: Write plume area - Updated on ---
# This block of code writes the plume area specific to each zone to the project folder
#######################

# Write Plumearea.csv file; exports plume m^2
arcpy.AddMessage("Writing file Plumearea.csv")
spamWriter = csv.writer(open(Folder + "\plume_area.csv", 'wb'), delimiter=',',
quotechar='|', quoting=csv.QUOTE_MINIMAL)

rows = arcpy.SearchCursor(Plume, "", "", "Plumearea") 
currentState = "" 

for row in rows: 
    if currentState != row.Plumearea: 
        currentState = row.Plumearea
    spamWriter.writerow([row.Plumearea])
arcpy.AddMessage("Done!")

# End new code 9.21.15 part 1

#######################
# Function: 
# This block of code 
#######################

#!!! START Write Tract_areas.csv Version 2!!!

#Latest running file appears to be CenShape_shp__2_

#Cen_Tracts = arcpy.GetParameterAsText(0)
#Plume = os.path.join(WEST_Root, "Plume1.shp")

#START#

#Plume = "C:\Users\tboe\Desktop\Temp\Version 4\Plume1.shp"
#arcpy.Delete_management(Plume)
#arcpy.CopyFeatures_management(Cen_Tracts, Plume)
#arcpy.MakeFeatureLayer_management (CenShape_shp__2_, Plume_Tract)

#CenShape_Clean = CenShape_shp__2_

#Clean_Censhape = os.path.join(Folder, "Clean_Censhape.shp")
#arcpy.CopyFeatures_management(CenShape_shp__2_, Clean_Censhape)

arcpy.AddField_management(CenShape_shp__2_, "Zone_1", "FLOAT", field_length = 50)
arcpy.AddField_management(CenShape_shp__2_, "Zone_2", "FLOAT", field_length = 50)
arcpy.AddField_management(CenShape_shp__2_, "Zone_3", "FLOAT", field_length = 50)

#Clean_Censhape1 = os.path.join(Folder, "Clean_Censhape1.shp")
#arcpy.CopyFeatures_management(Clean_Censhape, Clean_Censhape1)

#Start

#Clean_Censhape_old = os.path.join(Folder, "Clean_Censhape1.shp")
#arcpy.CopyFeatures_management(CenShape_shp__2_, Clean_Censhape_old)

Clean_Censhape = os.path.join(Folder, "Clean_Censhape.shp")

#arcpy.MakeFeatureLayer_management(Clean_Censhape_old, Clean_Censhape)
arcpy.MakeFeatureLayer_management(CenShape_shp__2_, Clean_Censhape)



val = "Level001"
field = arcpy.AddFieldDelimiters(Clean_Censhape, "LINE_ID")
selection = "{field} = '{val}'".format(field=field, val=val)
arcpy.SelectLayerByAttribute_management(Clean_Censhape, "NEW_SELECTION", selection)
arcpy.CalculateField_management(Clean_Censhape, "Zone_1", "!Area_m2b! / !SHAPE_Area!", "PYTHON", "#")

val = "Level002"
field = arcpy.AddFieldDelimiters(Clean_Censhape, "LINE_ID")
selection = "{field} = '{val}'".format(field=field, val=val)
arcpy.SelectLayerByAttribute_management(Clean_Censhape, "NEW_SELECTION", selection)
arcpy.CalculateField_management(Clean_Censhape, "Zone_2", "!Area_m2b! / !SHAPE_Area!", "PYTHON", "#")

val = "Level003"
field = arcpy.AddFieldDelimiters(Clean_Censhape, "LINE_ID")
selection = "{field} = '{val}'".format(field=field, val=val)
arcpy.SelectLayerByAttribute_management(Clean_Censhape, "NEW_SELECTION", selection)
arcpy.CalculateField_management(Clean_Censhape, "Zone_3", "!Area_m2b! / !SHAPE_Area!", "PYTHON", "#")

arcpy.SelectLayerByAttribute_management(Clean_Censhape, "CLEAR_SELECTION")

Plume_Dissolve = "Plume_Dissolve"
arcpy.Delete_management(Plume_Dissolve)
arcpy.Dissolve_management(Clean_Censhape, Plume_Dissolve,
                          ["GEOID10"], "Zone_1 SUM;Zone_2 SUM;Zone_3 SUM", "MULTI_PART", 
                          "DISSOLVE_LINES")

# Write file
arcpy.AddMessage("Writing file output.csv")
spamWriter = csv.writer(open(Folder + "\\tract_area.csv", 'wb'), delimiter=',',
quotechar='|', quoting=csv.QUOTE_MINIMAL)

rows = arcpy.SearchCursor(Plume_Dissolve, "", "", "GEOID10; SUM_Zone_1; SUM_Zone_2; SUM_Zone_3") 
currentState = "" 

for row in rows: 
    if currentState != row.GEOID10: 
        currentState = row.GEOID10
    spamWriter.writerow([row.GEOID10, row.SUM_Zone_1, row.SUM_Zone_2, row.SUM_Zone_3,])
arcpy.AddMessage("Done!")

##################
#!!! END Write Tract_areas.csv Version 2!!!
###################

# Delete temporary plumes; remove this section when troubleshooting
arcpy.AddMessage("Obliterating temporary plumes")
#arcpy.Delete_management(CenShape_shp)
#arcpy.Delete_management(CenShape_Sort)
arcpy.SelectLayerByAttribute_management(newlayer, "CLEAR_SELECTION")
arcpy.RefreshActiveView()
arcpy.AddMessage("Done!")

# !!!!!!!Start Image Section!!!!!!!

# Revert Plume variable back to Plume Shapefile; this zoom feature must reference a shapefile to function correctly
Plume = 'plume_clean_sort'

# Set local variables
arcpy.AddMessage("Loading local variables")
image = Folder
image += "\Zone1"
arcpy.AddMessage("Done!")

# Select feature Level001 in layer "CenShape_shp__2_"
arcpy.AddMessage("Selecting level: 001")
arcpy.SelectLayerByAttribute_management(Plume, "NEW_SELECTION", "\"LINE_ID\" = 'Level001'")
arcpy.AddMessage("Done!")

# Set map document to current; set data frame to default "layers"; define layer
arcpy.AddMessage("Setting data frame")
mxd=arcpy.mapping.MapDocument('CURRENT')
df=arcpy.mapping.ListDataFrames(mxd, "Layers")[0]
Layer=(mxd, Plume, df)[0]
layers = arcpy.mapping.ListLayers(mxd, "*", df)
arcpy.AddMessage("Done!")

# List names of layers to be automatically hidden; remove layers for clean images
names = ["Study Region Tract","Study Region Boundary",FileName,Plume_File,BarePlume,Plume,Zone_2,Zone_1]

arcpy.AddMessage("Disabling Layers")
# Turn off all layers listed in names list
for layer in layers:
  if layer.name in names:
    layer.visible = False
arcpy.AddMessage("Done!")

# Refresh TOC and Map
arcpy.RefreshTOC()
arcpy.RefreshActiveView()

# Add image layer based on end users selection in toolbox menu
if Imagery_Type == "ESRI Basemap Imagery":
    arcpy.AddMessage("Adding ESRI Basemap Imagery")
    Basemap_Imagery = arcpy.mapping.Layer(os.path.join(WEST_Root, "ESRI_Basemap_Imagery.lyr"))
else:
    arcpy.AddMessage("Adding Custom Imagery")
    Basemap_Imagery = arcpy.mapping.Layer(Imagery_Location)

# Add imagery layer
arcpy.mapping.AddLayer(df, Basemap_Imagery,"BOTTOM")
arcpy.AddMessage("Done!")

# Zoom to selected features and refresh view
arcpy.AddMessage("Zooming to selected feature")
df.zoomToSelectedFeatures()
arcpy.RefreshActiveView
arcpy.AddMessage("Done!")

# Capture image as .tif
arcpy.AddMessage("Capturing image")
arcpy.mapping.ExportToTIFF(mxd, image, df,
                           df_export_width=1066,
                           df_export_height=869,
                           tiff_compression="NONE",
                           geoTIFF_tags=True)
arcpy.AddMessage("Done!")

# Clip selected feature from captured image
arcpy.AddMessage("Clipping level 001 from captured image")
arcpy.Clip_management(image + ".tif", "", image + ".bmp", Plume, "255", "ClippingGeometry")
arcpy.AddMessage("Done!")

# Reset local variables
arcpy.AddMessage("Resetting local variables")
image = Folder
image += "\Zone2"
arcpy.AddMessage("Done")

# Select feature Level002 in layer "CenShape_shp__2_"
arcpy.AddMessage("Selecting level: 002")
arcpy.SelectLayerByAttribute_management(Plume, "NEW_SELECTION", "\"LINE_ID\" = 'Level002'")
arcpy.AddMessage("Done!")

# Zoom to selected features and refresh view
arcpy.AddMessage("Zooming to selected feature")
df.zoomToSelectedFeatures()
arcpy.RefreshActiveView
arcpy.AddMessage("Done!")

# Capture image as .tif
arcpy.AddMessage("Capturing image")
arcpy.mapping.ExportToTIFF(mxd, image, df,
                           df_export_width=1066,
                           df_export_height=869,
                           tiff_compression="NONE",
                           geoTIFF_tags=True)
arcpy.AddMessage("Done!")

# Clip selected feature from captured image
arcpy.AddMessage("Clipping level 002 from captured image")
arcpy.Clip_management(image + ".tif", "", image + ".bmp", Plume, "255", "ClippingGeometry")
arcpy.AddMessage("Done!")

# Reset local variables
arcpy.AddMessage("Resetting local variables")
image = Folder
image += "\Zone3"
arcpy.AddMessage("Done!")

# Select feature Level003 in layer "CenShape_shp__2_"
arcpy.AddMessage("Selecting level: 003")
arcpy.SelectLayerByAttribute_management(Plume, "NEW_SELECTION", "\"LINE_ID\" = 'Level003'")
arcpy.AddMessage("Done!")

# Zoom to selected features and refresh view
arcpy.AddMessage("Zooming to selected feature")
df.zoomToSelectedFeatures()
arcpy.RefreshActiveView
arcpy.AddMessage("Done!")

arcpy.AddMessage("Capturing image")
# Capture image as .tif
arcpy.mapping.ExportToTIFF(mxd, image, df,
                           df_export_width=1066,
                           df_export_height=869,
                           tiff_compression="NONE",
                           geoTIFF_tags=True)
arcpy.AddMessage("Done!")

# Clip selected feature from captured image
arcpy.AddMessage("Clipping level 003 from captured image")
arcpy.Clip_management(image + ".tif", "", image + ".bmp", Plume, "255", "ClippingGeometry")
arcpy.AddMessage("Done!")

if arcpy.CheckExtension("Spatial") == "Available":
    arcpy.AddMessage("Spatial Analyst license found. Proceeding with image analysis")
else:
    arcpy.AddWarning("WARNING: Spatial Analyst license not found. WEST requires Spatial Analyst to automatically identify outdoor surfaces. A default estimate has been applied. You may tweak the default values within the WEST calculator.")

# Clear selectio before finishing
arcpy.SelectLayerByAttribute_management(Plume, "CLEAR_SELECTION")

arcpy.AddMessage("Done!")

#####

# Set local variables
inRaster1 = os.path.join(Working_Folder_Project, "Zone1.bmp")
inRaster2 = os.path.join(Working_Folder_Project, "Zone2.bmp")
inRaster3 = os.path.join(Working_Folder_Project, "Zone3.bmp")
sigFile = os.path.join(WEST_Root, "061818v1.gsg")
probThreshold = "0.0"
aPrioriWeight = "EQUAL"
aPrioriFile = ""
outConfidence = ""

# Check out license
arcpy.CheckOutExtension("Spatial")

arcpy.AddMessage('Starting image classification process. This may take a few minutes...')
                                   
# Zone 1 Classification
arcpy.AddMessage('Classifying Zone 1')
mlcOut = MLClassify(inRaster1, sigFile, probThreshold, aPrioriWeight, 
                    aPrioriFile, outConfidence)
mlcOut.save(os.path.join(Working_Folder_Project, "Zone1.img"))
arcpy.AddMessage('Done!')

# Zone 2 Classification
arcpy.AddMessage('Classifying Zone 2')
mlcOut = MLClassify(inRaster2, sigFile, probThreshold, aPrioriWeight, 
                    aPrioriFile, outConfidence)
mlcOut.save(os.path.join(Working_Folder_Project, "Zone2.img"))
arcpy.AddMessage('Done!')

# Zone 3 Classification
arcpy.AddMessage('Classifying Zone 3')
mlcOut = MLClassify(inRaster3, sigFile, probThreshold, aPrioriWeight, 
                    aPrioriFile, outConfidence)
mlcOut.save(os.path.join(Working_Folder_Project, "Zone3.img"))
arcpy.AddMessage('Done!')

#add new field to .img files for storing pixel countrs

inFeatures1 = os.path.join(Working_Folder_Project, "Zone1.img")
fieldName1 = "Zone1"
fieldPrecision = 1
inFeatures2 = os.path.join(Working_Folder_Project, "Zone2.img")
fieldName2 = "Zone2"
inFeatures3 = os.path.join(Working_Folder_Project, "Zone3.img")
fieldName3 = "Zone3"

arcpy.AddField_management(inFeatures1, fieldName1, "Double", fieldPrecision, "", "")
arcpy.AddField_management(inFeatures2, fieldName2, "Double", fieldPrecision, "", "")
arcpy.AddField_management(inFeatures3, fieldName3, "Double", fieldPrecision, "", "")

#specify number of zones and surface types; set default values to 0

Zone1_list = [0,0,0]
Zone2_list = [0,0,0]
Zone3_list = [0,0,0]
Compatible_Surfaces = ['Soil', 'Concrete', 'Asphalt', 'Vegetation', 'Water']

#process zone 1; store values to list

arcpy.AddMessage('Processing Zone 1 results')
row_total = 0
fields_in_cursor = ["Count", "Zone1", "CLASSNAME"]
with arcpy.da.UpdateCursor(os.path.join(Working_Folder_Project, "Zone1.img"), fields_in_cursor) as cursor:
    for row_field_Name in cursor:
            row_total += row_field_Name[0]
            cursor.updateRow(row_field_Name)           
with arcpy.da.UpdateCursor(os.path.join(Working_Folder_Project, "Zone1.img"), fields_in_cursor) as cursor:            
    for row_field_Name in cursor:
            row_field_Name[1] = row_field_Name[0] / row_total
            cursor.updateRow(row_field_Name)
#align surface % with file schema
            if row_field_Name[2] in Compatible_Surfaces[0]: Zone1_list[0] = row_field_Name[1]
            if row_field_Name[2] in Compatible_Surfaces[1]: Zone1_list[1] = row_field_Name[1]
            if row_field_Name[2] in Compatible_Surfaces[2]: Zone1_list[2] = row_field_Name[1]
            #note that this next row adds vegetation results to soil since WEST doesn't account for vegetation yet; remove plus to fix this
            if row_field_Name[2] in Compatible_Surfaces[3]: Zone1_list[0] += row_field_Name[1]
            #the next row is turned off since WEST doesn't account for water. The % is still taken from the total. To account for water, expand the list (add another zero to zone list, add water to compatible surfaces and remove the comment below
            #if row_field_Name[2] in Compatible_Surfaces[4]: Zone1_list[4] = row_field_Name[1]                  
arcpy.AddMessage('Done!')

#process zone 2; store values to list
            
arcpy.AddMessage('Processing Zone 2 results')
row_total = 0
fields_in_cursor = ["Count", "Zone2", "CLASSNAME"]
with arcpy.da.UpdateCursor(os.path.join(Working_Folder_Project, "Zone2.img"), fields_in_cursor) as cursor:
    for row_field_Name in cursor:
            row_total += row_field_Name[0]
            cursor.updateRow(row_field_Name)         
with arcpy.da.UpdateCursor(os.path.join(Working_Folder_Project, "Zone2.img"), fields_in_cursor) as cursor:            
    for row_field_Name in cursor:
            row_field_Name[1] = row_field_Name[0] / row_total
            cursor.updateRow(row_field_Name)
#align surface % with file schema
            if row_field_Name[2] in Compatible_Surfaces[0]: Zone2_list[0] = row_field_Name[1]
            if row_field_Name[2] in Compatible_Surfaces[1]: Zone2_list[1] = row_field_Name[1]
            if row_field_Name[2] in Compatible_Surfaces[2]: Zone2_list[2] = row_field_Name[1]
            #note that this next row adds vegetation results to soil since WEST doesn't account for vegetation yet; remove plus to fix this
            if row_field_Name[2] in Compatible_Surfaces[3]: Zone2_list[0] += row_field_Name[1]
            #the next row is turned off since WEST doesn't account for water. The % is still taken from the total. To account for water, expand the list (add another zero to zone list, add water to compatible surfaces and remove the comment below
            #if row_field_Name[2] in Compatible_Surfaces[4]: Zone1_list[4] = row_field_Name[1]
arcpy.AddMessage('Done!')

#process zone 3; store values to list
            
arcpy.AddMessage('Processing Zone 3 results')
row_total = 0
fields_in_cursor = ["Count", "Zone3", "CLASSNAME"]
with arcpy.da.UpdateCursor(os.path.join(Working_Folder_Project, "Zone3.img"), fields_in_cursor) as cursor:
    for row_field_Name in cursor:
            row_total += row_field_Name[0]
            cursor.updateRow(row_field_Name)           
with arcpy.da.UpdateCursor(os.path.join(Working_Folder_Project, "Zone3.img"), fields_in_cursor) as cursor:            
    for row_field_Name in cursor:
            row_field_Name[1] = row_field_Name[0] / row_total
            cursor.updateRow(row_field_Name)
# align surface % with file schema
            if row_field_Name[2] in Compatible_Surfaces[0]: Zone3_list[0] = row_field_Name[1]
            if row_field_Name[2] in Compatible_Surfaces[1]: Zone3_list[1] = row_field_Name[1]
            if row_field_Name[2] in Compatible_Surfaces[2]: Zone3_list[2] = row_field_Name[1]
            #note that this next row adds vegetation results to soil since WEST doesn't account for vegetation yet; remove plus to fix this
            if row_field_Name[2] in Compatible_Surfaces[3]: Zone3_list[0] += row_field_Name[1]
            #the next row is turned off since WEST doesn't account for water. The % is still taken from the total. To account for water, expand the list (add another zero to zone list, add water to compatible surfaces and remove the comment below
            #if row_field_Name[2] in Compatible_Surfaces[4]: Zone1_list[4] = row_field_Name[1]
arcpy.AddMessage('Done!')


#multiply by 100 for whole number

Zone1_list = [i * 100 for i in Zone1_list]
Zone2_list = [i * 100 for i in Zone2_list]
Zone3_list = [i * 100 for i in Zone3_list]

#reconfigure lists to match required WEST inputs. This can be modified for future compatibility

image_anaylsis_line1 = ['','Zone1', 'Zone2', 'Zone3']
image_anaylsis_line2 = ['Asphalt (%)',Zone1_list[2],Zone2_list[2],Zone3_list[2]]
image_anaylsis_line3 = ['Concrete (%)',Zone1_list[1],Zone2_list[1],Zone3_list[1]]
image_anaylsis_line4 = ['Soil (%)',Zone1_list[0],Zone2_list[0],Zone3_list[0]]

arcpy.AddMessage('Surface Analysis Results:')
arcpy.AddMessage(str(image_anaylsis_line1))
arcpy.AddMessage(str(image_anaylsis_line2))
arcpy.AddMessage(str(image_anaylsis_line3))
arcpy.AddMessage(str(image_anaylsis_line4))

#write ground surface data file

arcpy.AddMessage('Writing ground surface data file...')
with open(os.path.join(Working_Folder_Project, 'ground_surface_data.csv'), 'wb') as myfile:
    wr = csv.writer(myfile, quotechar='|', quoting=csv.QUOTE_MINIMAL)
    wr.writerow(image_anaylsis_line1)
    wr.writerow(image_anaylsis_line2)
    wr.writerow(image_anaylsis_line3)
    wr.writerow(image_anaylsis_line4)

# The End

