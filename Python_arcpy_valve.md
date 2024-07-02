Valve Placement Python Code
The purpose of this project is to place valve symbols based on distance from curb line.  
### Summary of Code
* Join Service feature with valve distance table by service number ID, keeping only matching rows
* Calculate the Service's angles and rotate counterclockwise with East as 0 degrees.
* Retain only the parts of the services within the curblines, that is within the street block not in the street.
    * Fill the curblines to make a curbline polygon
    * Intersect the polygon with services. This may split services into more than one polyline if the service streches across two ploygons. It starts at the block across the street. This will produce two or more valves for the same service if ran without a correction 
    * Later, correction will come
* Extend the service polylines to touch the curb, if they already do not. Use arcpy.ExtendLine_edit()
    * Buffert distance set to 30 FEET. Make larger if necessary. 
        * Later, the curb_start field will specify whether polyline touches curbline with 'na' value. Inspect them
    * This takes very long to run. Run overnight.
    * This requires the service and curblines to be in same feature
    * use arcpy.Merge_management() to merge into same feature
    * Erase the curblines using arcpy.Erase_analysis() from the feature leaving only the extended service polylines
* There will be multipart services. Use arcpy.Dissolve_management() to combine all touching services to rid multiparts. 
    * spatial join them back to the service join feature to reobtain the attribute table including the valve information
* Assign the services a curb ID, determined by which curb it touches
    * dissolve the curblines to reduce number of multipart polylines in curbs, reduces processing time
    * assign the curb ID to be the row's object ID
    * This will help decrease time in finding service polyline start touches curbline.
    * Spatial join the curb ID to the service polyline for the services to obtain the curb ID
* Make a point feature of the start of the service polyline
    * Take the X start and Y start, and store it into a point feature with the address and service number
    * This will point determines if the service polyline starts at the curbline
        * the valve distance moves from the curb, so the polyline must start at the curb for the valve to be placed correctly
* Evaluate that point with .disjoint() to see if touches curbline and store column . If touches, then curb_start is true
   * if disjoint is false, curb_start is true, and vice versa
   * insert the curb_start column to the service polyline
* Find the position of valve along service polyline with .positionAlongLine()
   * using a function
   * use length minus distance is curb start is false
* Rid extra polyline
   * Intersection of curb polygon and services can cut services into mulptiple pieces if the service starts inside another curb polygon.
   * Keep the service polyline of the longest length. This highly likely to be the one from curb to premise.
   * 
   
   
    
### Packages
The code uses the following packages: 
```python
    import arcpy, os, math, pandas as pd, numpy as np, collections
```
### Set up
Set the work space. 
Join valve excel and P_service in ArcGIS prior to running script. uncheck keep all target features. Export it as separate feature. Name it 'Service_join'

```python
    arcpy.env.overwriteOutput=True
    arcpy.env.workspace = "S:/FieldOps/Oper. System Support/Intern 2019/02_Service valve placement/EntireCityPro/EntireCityPro.gdb"
    service = 'Service_join'
```

### Angle Calculation 
Create fields in 'Service_join' to store the X and Y coordinates of the start and end of each service polyline.  
```python
    fc = service 
    arcpy.AddField_management(fc, "QUADRANT", "STRING")
    
    fields = ["Y_START", "Y_END", "X_START", "X_END", "ANGLE_ADJ", "ANGLE_1" ]
    for field in fields:
        arcpy.AddField_management(fc, str(field), "DOUBLE")
    
    with arcpy.da.UpdateCursor(fc, ["Y_START", "Y_END", "X_START", "X_END", "SHAPE@"]) as cursor:
        for row in cursor:
            row[0] = row[4].firstPoint.Y
            row[1] = row[4].lastPoint.Y
            row[2] = row[4].firstPoint.X
            row[3] = row[4].lastPoint.X
            cursor.updateRow(row)
```
Angle Calculation using Arctangent from the X and Y starts and ends. The angles are adjusted to be based on angle = 0 degrees ( x > 0, y = 0) and rotating counter clockwise
   

```python 
    length = []
     angle = []
     quad = []
     angle_adj = []
           
    with arcpy.da.UpdateCursor(fc, ["Y_START", "Y_END", "X_START", "X_END", "ANGLE_ADJ", "Shape_Length","QUADRANT", "ANGLE_1" ]) as cursor:
        for row in cursor:
             # Access and print the row values by index position.
             #   Y_START: row[0]
             #   Y_END: row[1]
             #   X_START: row[2]
             #   X_END: row[3]
             #   ANGLE_ADJ: row[4]
             #   LENGTH: row[5]
             #   QUADRANT: row[6]
             #   ANGLE_1: row[7]
             #print(row[4])
            
             #length_ = math.sqrt(((row[3]-row[2])**2)+((row[1]-row[0])**2))
             length_ = row[5]
             angle_ = math.degrees(math.atan(abs(row[0]-row[1])/abs(row[2]-row[3])))  
             length.append(length_)
             angle.append(angle_)  
             row[7] = angle_
             if (row[2] < row[3]) and (row[0] < row[1]):
                 quad.append(1)
                 row[6] = 'i'
                 row[4] = angle_ + 0
             elif (row[2] > row[3]) and (row[0] < row[1]):
                 quad.append('ii')
                 row[6] = 'ii'
                 row[4] = 180 - angle_ 
             elif (row[2] > row[3]) and (row[0] > row[1]):
                 quad.append('iii')
                 row[6] = 'iii'
                 row[4] = angle_ + 180
             elif (row[2] < row[3]) and (row[0] > row[1]):
                 quad.append('iv')
                 row[6] = 'iv'
                 row[4] = 360 - angle_ 
             cursor.updateRow(row)
``` 


### Fill in Curblines to make polygons

```python
    fc = "Conflation_curblines"
    out_feature_class = "Curb_fill"
    arcpy.FeatureToPolygon_management(fc,out_feature_class , "", "ATTRIBUTES", "")
```
### Keep the parts of the services inside the Curblines polygons
```python
    inFeatures = ["Curb_fill", service]
    intersectOutput = "Service_curb"
    clusterTolerance = ""    
    arcpy.Intersect_analysis(inFeatures, intersectOutput, "", clusterTolerance, "INPUT")
```
### Place Curblines and Intersected Services in same Feature
```python
    curb = "Conflation_curblines"
    target_features = ["Service_curb",  fc]
    out_feature_class = "Service_curb_merge"
    arcpy.Merge_management(target_features, out_feature_class)
```
### Extend the Merged Services to Curbline 
This runs very long time. Any services within 20 Ft will be extended to curbline. adjust the extension distance if needed. The services and curblines must be in one feature together.
```python
    arcpy.ExtendLine_edit("service_curb_merge", "20 Feet", "FEATURE")
```

### Erase the Curblines
Only the extended services remain.
```python
    in_features = "Service_curb_merge"
    erase_features = curb
    out_feature_class = "Service_curb_extend"
    xy_tolerance = ""
    arcpy.Erase_analysis(in_features, erase_features, out_feature_class, xy_tolerance)
```

### Dissolve the Service Curb Extends
Eliminates multi-part polyline services
```python
    in_features = "Service_curb_extend"
    out_feature_class = "Service_curb_extend_dissolve"
    arcpy.Dissolve_management(in_features, out_feature_class, "", "","MULTI_PART", "UNSPLIT_LINES")
```

### Spatial Join
The dissolved polylines takes back the attribute table of the services 
```python
    target_features = "Service_curb_extend_dissolve"
    join_features = service
    out_feature_class = "Service_curb_extend_dissolve_join"
    arcpy.SpatialJoin_analysis(target_features, join_features, out_feature_class)
```
### Curbline ID
Identifies if the polyline start intersects the curbline faster.
Dissolve the curblines:

```python
    in_features = "Conflation_curblines"
    out_feature_class = "curb_dissolve"
    arcpy.Dissolve_management(in_features, out_feature_class, "", "", 
                              "MULTI_PART", "UNSPLIT_LINES")
```
Assign the ObjectID as Curb ID
```python
    arcpy.AddField_management(out_feature_class, "curb_id", "STRING")
    fields = ["OID@", "curb_id"]
    with arcpy.da.UpdateCursor(out_feature_class, fields) as cursor:
        for row in cursor:
            row[1]=(row[0])
            cursor.updateRow(row)
```
Spatial join the dissolved curbline to give the Curb ID to the extended polyline service
```python
    fc = "Service_curb_extend_dissolve_join_curbid"
    target_features = "Service_curb_extend_dissolve_join"
    join_features = "curb_dissolve"
    out_feature_class = "Service_curb_extend_dissolve_join_curbid"
    arcpy.SpatialJoin_analysis(target_features, join_features, out_feature_class)
```
### Finding start point of polyline service 
Find the X and Y starts of the polyline service
```python
    x_start_poly = []
    y_start_poly = []
    address = []
    service_num = []
    curb_id = []
       
    fields = ["SHAPE@","P_Service_SERVICE_NUMBER_ID","P_Service_ADDRESS", "curb_id"]
    with arcpy.da.SearchCursor(fc, fields) as cursor:
        for row in cursor:
                #position = row[0].positionAlongLine(float(row[1]), False)
                #valve_position.append(position)
                address.append(row[2])
                service_num.append(row[1])
                curb_id.append(row[3])
                x_start_poly.append(row[0].firstPoint.X)
                y_start_poly.append(row[0].firstPoint.Y)
```
Create point feature of the start point
```python
    out_path = "S:/FieldOps/Oper. System Support/Intern 2019/02_Service valve placement/EntireCityPro/EntireCityPro.gdb"
    out_name = "poly_start3"
    geometry_type = "POINT"
    template = ""
    has_m = "DISABLED"
    has_z = "DISABLED"
    
        # Use Describe to get a SpatialReference object
    spatial_reference = arcpy.Describe(service).spatialReference
    
        # Execute CreateFeatureclass
    arcpy.CreateFeatureclass_management(out_path, out_name, geometry_type, template, has_m, has_z, spatial_reference)
    arcpy.AddField_management(out_name, "address", "STRING")
    arcpy.AddField_management(out_name, "service_num", "STRING")
    arcpy.AddField_management(out_name, "curb_id", "STRING")
    arcpy.AddField_management(out_name, "curb_start", "STRING")
        
        #fill poly_start
    poly_start_list = list(zip(x_start_poly, y_start_poly, service_num, address, curb_id ))
    fields =["SHAPE@X", "SHAPE@Y", "service_num", "address", "curb_id"]
    
    cursor = arcpy.da.InsertCursor(out_name, fields)
    
    for row in poly_start_list:
        cursor.insertRow(row)
```

### Does Polyline Service extend start at curbline?
 Finding which service polylines start at curb
Write a function to be used in for loop for determining 
  Indicates if the base and comparison geometries share no points in common.
   Two geometries intersect if disjoint returns False.
   True = 1 and False = 0. 
   The function takes one polyline and determines if it touches the curbline associated with its curb ID. If touches, touch_sum is 0, else it is 1. The length of the list of curblines is always 1.

```python    
    def poly_start (polyline, curblines):
        touches = []
        len_curblines = len(curblines)
        for curb_ in curblines:
            touch = polyline.disjoint(curb_)
            if touch == False:
                break 
            touches.append(touch)
        touch_sum = sum(touches)
        if touch_sum >= len_curblines:
            #arcpy.FlipLine_edit(row[0])
            curbstart = 'false'
            return curbstart
        elif touch_sum < len_curblines:
            curbstart = 'true'
            return curbstart
```

Read in the curb and polyline service to Python 
    
```python    
    curb_geom =[]
    curb_dissolve_curb_id = []        
    fields = ["SHAPE@", "curb_id"]
    with arcpy.da.SearchCursor("curb_dissolve", fields) as cursor:
        for row in cursor:
                curb_geom.append(row[0])
                curb_dissolve_curb_id.append(row[1])
    
    poly_geom = []   
    with arcpy.da.SearchCursor(out_name, "SHAPE@") as cursor:
        for row in cursor:
            poly_geom.append(row[0])
        #another way to get feature geom
    #curb_geom = arcpy.CopyFeatures_management("curb_dissolve", arcpy.Geometry())    
    curb_geom_id = list(zip(curb_geom,curb_dissolve_curb_id))
```
Use Pandas to select the curbline from curb ID
```python
    curb_start = []
    curb_dic = {'curb_id':curb_dissolve_curb_id, 'curb_geom':curb_geom}
    curb_pd = pd.DataFrame(curb_dic)
    
    fields = ["SHAPE@", "curb_start", "curb_id"]
    with arcpy.da.UpdateCursor(out_name, fields) as cursor:
       for row in cursor:
           if row[2] != None:
               curb__ = list(curb_pd.curb_geom[curb_pd.curb_id == str(row[2])]) #curb_id mus be string for pandas
               row[1] = poly_start(row[0], curb__)
           elif row[2] == None:
               row[1] = 'na'
           curb_start.append(str(row[1]))
           cursor.updateRow(row)
```
Add  field curb start to polyline attribute table to store whether polyline starts at curbline
```python
    fc = "Service_curb_extend_dissolve_join_curbid"
    arcpy.AddField_management(fc, "curb_start", "STRING")
    
    curb_start_list = list(zip(curb_start))
    cursor = arcpy.da.InsertCursor(fc, "curb_start")
    i =0
    with arcpy.da.UpdateCursor(fc, "curb_start") as cursor:
        for row in cursor:
            row[0]=curb_start[i]
            i = i +1
            cursor.updateRow(row)
```
### Valve Position
Define function to find valve position using .positionAlongLine. If the service polyline start is not at the curbline, then the position is taken from the difference of (polyline's length) - (Valve Distance) 
```python
    def valve_pos(shape, dist, start, length):
        if start == 'true':
            position = shape.positionAlongLine(float(dist), False)
            return position
        elif start == 'false':
            diff = float(length) - float(dist)
            position = shape.positionAlongLine(float(diff), False)
            return position
```
Initialize the variables
```python
    valve_position = []
    dist_valve = []
    ref_valve = []
    service_num = []
    address = []
    angle = []
    angle_adj = []
    quad = []
    shape_length = []
    angle_adj = [] 
```
Read attribute table into Python and finding valve position based on data
```python
    fields = ["SHAPE@", "Service_Valve_20190614__DIST_VALVE", "Service_Valve_20190614__VALVE_LOC_REF","P_Service_SERVICE_NUMBER_ID","P_Service_ADDRESS", "curb_start", "Shape_Length", "ANGLE_ADJ"]
    with arcpy.da.SearchCursor(fc, fields) as cursor:
        for row in cursor:
    #        #"SHAPE@", row[0]
    #        "DIST_VALVE", row[1]
    #        "Sheet1__VALVE_LOC_REF", row[2]
    #        "SERVICE_NUM", row[3]
    #        "P_Service_ADDRESS", row[4]
    #        "P_Service_ANGLE_1", row[5]
    #        "P_Service_ANGLE_ADJ", row[6]
    #        "P_Service_QUADRANT" row[7]
    
                position = valve_pos(row[0], row[1], row[5], row[6])
                valve_position.append(position)
                dist_valve.append(row[1])
                ref_valve.append(row[2])
                service_num.append(row[3])
                address.append(row[4])
                shape_length.append(row[6])
                angle_adj.append(row[7])
```

### Stores Index with False Curb_start
```python
q = 0
false_ind = []
for curb in curb_start:
    if curb == "false":
        false_ind.append(q)
    q = q + 1
```
### Replaces mistyped Valve distances with distance 0
Valves can be manually corrected later. about 6 are mistyped. This also prints the address of mistyped distances
```python
q = 0
dist_str_ind = []
for dist in dist_valve:
    if dist == '#VALUE!':
        dist_str_ind.append(q)
    q = q + 1
    
for ind in dist_str_ind:
    print(address[ind], service_num[ind], dist_valve[ind])
    dist_valve[ind] = 0
```

### Eliminate Duplicated Polylines
The curblines retain only the part of services located within the curblines, and as a result when service polylines start on the otherside of the block, two or more polylines will exist of for the same service. This part will eliminate the shorter of the two because the the longer polyline in most of the cases is the part from the curbline to the premise. 
#
Column bind the lists into a dataframe  
```python
    poly_many_address = []    
    poly_many_service = []  
    poly_many_count = []          
    
    address_poly_list = list(zip(address, service_num))
```  
The collections.Counter count the occurrences of service numbers. Ideally, the service number occurs once meaning the curbline did not bisect the service polyline  
```python  
    address_poly_count=collections.Counter(address_poly_list).most_common()
    
    for i in range(0,len(address_poly_count)):
        poly_many_address.append(address_poly_count[i][0][0])
        poly_many_service.append(address_poly_count[i][0][1])
        poly_many_count.append(address_poly_count[i][1])
            
    poly_many_list = list(zip(poly_many_address, poly_many_service, poly_many_count))
```
Make Pandas dataframe of previous read-in lists   
```python              
    col_pd = {'address':address, 'service_num':service_num, 'valve_position':valve_position, 'dist_valve':dist_valve, 'angle_adj':angle_adj, 'length':shape_length}
    
    valve_pd = pd.DataFrame(col_pd)
```
#
Storing all the Service Numbers that occur in the polyline start. This will search for multiple polylines of the same service. 

```python   
 service_num_poly=[]
    
    for i in range(0, len(address_poly_count)):
        service_num_poly.append(address_poly_count[i][0][1])
  ```
 #
```python    
    #list of pandas rows
    test_address = []
    test_service = []
    test_valve_position = []
    test_dist_valve = []
    test_angle_adj = []
    test_length = []
    
    col_names =["valve_position", "dist_valve", "service_num", "address", "angle_adj", "length"]
    col_lists = [test_valve_position, test_dist_valve, test_service, test_address, test_angle_adj, test_length]
    # use list() to take out of pandas
     
     #use q to start the for loop to append 
    q = 0
    for i in range(0, len(service_num_poly)):
        #test.append(list(valve_pd.loc[valve_pd['service_num']== service_num_poly[i]].max()))
        test_row = valve_pd[(valve_pd.length==valve_pd.length[valve_pd.service_num == service_num_poly[i]].max()) & (valve_pd.service_num == service_num_poly[i]) ]
        test_index = list(test_row.index)
        q = q +1
        for j in range(0, len(col_names)):
            col_lists[j].append(list(test_row.loc[test_index, col_names[j]])[0])
```
### Create Valve Feature and Insert lists from a top
```python
    out_path = "S:/FieldOps/Oper. System Support/Intern 2019/02_Service valve placement/EntireCityPro/EntireCityPro.gdb"
    out_name = "valve"
    geometry_type = "POINT"
    template = ""
    has_m = "DISABLED"
    has_z = "DISABLED"
    
    # Use Describe to get a SpatialReference object
    spatial_reference = arcpy.Describe("S:/FieldOps/Oper. System Support/Intern 2019/02_Service valve placement/PGW_RAMTECH_0715.gdb/P_valve").spatialReference
    
    # Execute CreateFeatureclass
    arcpy.CreateFeatureclass_management(out_path, out_name, geometry_type, template, has_m, has_z, spatial_reference)
```    
    
 ```python   
    # insert field names
    fc = "valve"
    
    arcpy.AddField_management(fc, "dist_valve", "FLOAT")
    arcpy.AddField_management(fc, "service_num", "STRING")
    arcpy.AddField_management(fc, "address", "STRING")
    arcpy.AddField_management(fc, "angle_adj", "FLOAT")
    arcpy.AddField_management(fc, "poly_length", "FLOAT")
    arcpy.AddField_management(fc, "curb_start", "STRING")
    
    test_list = list(zip(test_valve_position, test_dist_valve, test_service, test_address, test_angle_adj, test_length, curb_start ))
    fields =["SHAPE@", "dist_valve", "service_num", "address", "angle_adj", "poly_length", "curb_start"]
    
    cursor = arcpy.da.InsertCursor("S:/FieldOps/Oper. System Support/Intern 2019/02_Service valve placement/EntireCityPro/EntireCityPro.gdb/valve3", fields)
    
    for row in test_list:
        cursor.insertRow(row)
```
