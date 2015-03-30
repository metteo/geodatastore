### Introduction ###
GeoDataStore is a sample application built on top of App Engine. It is designed to show you how easy it is to create a flexible, accessible geo database on top of App Engine. GeoDataStore allows you to retrieve data in both JSON and KML format, and it includes a map-based administrative interface for viewing, editing, and adding data using HTTP GET/POST operations. The code is open source, so you can take it and create your own code based on it.


### Why an App Engine application? ###
There are many different kinds of geo databases out there, some open source, some commercial proprietary systems. On the Open Source end you could use standard non-geo databases, such as MySQL, to store geo data, or use a geo database such as PostGIS. Many proprietary databases also come with geographic components, such as Microsoft's SQL Server, and Oracle's Database products. And there are also commercial specialty database servers, such as Google's Fusion server and ESRI's ArcGIS Server.

All these databases have one thing in common: They must be hosted, either within your own hosting environment or one you rent from somewhere else. And if you need to scale support for them as your application grows in popularity, that can mean many hours spent optimizing code that could be better spent elsewhere.

Running GeoDataStore, or some other geo application that create (?), allows you to build an application that will scale well without additional programming on your part (weird sentence). It is built on Google's infrastructure, and on the backend, uses many of the same techniques Google does for storing and querying data. Plus, it's easy to develop for (?).

### Features ###
GeoDataStore is written all in Python and Javascript, and uses the simple App Engine Datastore and Users APIs (link) and the webapp framework for hosting. The Datastore allows for easy querying and storing of the data. The User API handles authentication and storing of user ids. The webapp framework allows use to easily separate the front-end from the back end.

GeoDataStore stores simple geo data. It stores point, line, and polygon data, and allows you to give each a name, description, and as many tags as you want. Queries are by type or user id. It serves data as KML or JSON, or through the built in map interface. Data is entered into the data store through HTTP Get or Post methods.

The default front end interface uses the Google Maps API and allows you to view, add, edit, and delete data. It allows you to create new markers via map clicks, local searches, or geocodes, and then continue to edit the markers by dragging and changing the textual information.

### The Datastore ###

The data is stored in a model called Geometry, and has the following structure:

```
Model
  name = a string
  description = a string, can hold HTML
  type = a string, point, line, or poly
  dateModified = the date when last modified
  coordinates = a list of GeoPts, which contain lat and lng
  bound = bounding box, a list containing east, west, north, south floats
  timeStamp = the when first added
  altitudes = A list of altitudes as float
  userId = if user logged in, the userid that created it
  tags = A string list of tags
```

Here's what that looks like in code:

```
class Geometry(db.Model):
  name = db.StringProperty()
  description = db.StringProperty(multiline=True)
  type = db.StringProperty()
  dateModified = db.DateProperty(auto_now=True)
  coordinates = db.ListProperty(db.GeoPt, default=None)
  bound = db.ListProperty(float, default=None)
  timeStamp = db.DateProperty(auto_now_add=True)
  altitudes = db.ListProperty(float, default=None)
  userId = db.StringProperty(default=None)
  tags = db.ListProperty(unicode,default=None)
```

To query the datastore, we only have to call the gql method of Geometry: geometries = Geometry.gql(qryString). That returns an iterator over a list of Geometry entities which you can then process. For instance:
```
    geometries = Geometry.gql(qryString)
    outputAction = {'json': jsonOutput(geometries,'get'),'kml': kmlOutput(geometries)}
```

### WebApp RequestHandler ###
GeoDataStore accepts both Get and Post requests, and essentially treats them the same. Post, of course, allows sending larger amounts of data and for shorter URLs, and is more secure. We decided to treat them the same, that is to allow adding data using Get, to allow for easy user testing of the datastore without using the Maps interface. These are the steps involved:

All operations go through the same RequestHandler: Request.

  * Request receives the request, which should include an operations parameter, and sends it to operationPicker.
  * Based on the operations parameter, operationPicker calls the appropriate CRUD method, addGeometries, getGeometries, editGeometries, or deleteGeometries.
  * Each operation conducts its business, and then returns a content and a contentType variable which specifies the MIME type to be used. That way the operationPicker method doesn't need to know anything about each operation.
  * It then writes out the response using self.response.out.write, after adding in the Content-Type header.

```
class Request(webapp.RequestHandler):
  def post(self):
    self.operationPicker()

  def get(self):
    self.operationPicker()

  def operationPicker(self):
      operation = self.request.get('operation')
      out,contentType = '',''
      if operation == 'add':
        out,contentType = self.addGeometries()
      elif operation == 'edit':
        out,contentType = self.editGeometries()
      elif operation == 'delete':
        out,contentType = self.deleteGeometries()
      else:
        out,contentType = self.getGeometries()
      self.response.headers.add_header('Content-Type', contentType)
      self.response.out.write(out)
```

At this point, the edit operation is dense. You can't edit a geometry without passing everything you know about the geometry back to GeoDataStore. This mirrors the Google Data Protocols (link), and the Atom Publishing Protocol (link) they implement, which similarly require full edits.

### Bounding Box Queries ###
When each geometry is created or edited, GeoDataStore calculates a bounding box (BBOX) for the it, the maximum North, Sourth, East, and West values, and stores them in the geometry entity. This is useful for doing bounding box queries, or a "show me everything in this area" type of query. Currently, GeoDataStore only supports BBOX queries in KML creation.

A BBOX parameter is formated like this: BBOX=[bboxWest](bboxWest.md),[bboxSouth](bboxSouth.md),[bboxEast](bboxEast.md),[bboxNorth](bboxNorth.md). For instance, you might give this request:

`/gen/request?operation=get&BBOX=-1,-1,1,1`

which would request everything in the box 1 degree west, 1 degree south, 1 degree east, 1 degree north. The BBOX parameter designed to match the format used by KML, to allow easy integration with KML NetworkLinks. The following NetworkLink:

```
<?xml version="1.0" encoding="UTF-8"?>
<kml xmlns="http://earth.google.com/kml/2.2">
  <NetworkLink>
    <Link>
      <href>http://!GeoDataStore.prom.corp.google.com/gen/request</href>
      <viewRefreshMode>onStop</viewRefreshMode>
      <httpQuery>operation=get;output=kml</httpQuery>
    </Link>
  </NetworkLink>
</kml>
```

would send the following request:

`http://!GeoDataStore.appspot.com/gen/request?operation=get&output=kml&BBOX=-1,-1,1,1`

The BBOX parameter is automatically added because the 

&lt;viewRefreshMode&gt;

 is set to onStop.

When GeoDataStore receives a request, it checks for the BBOX parameter. If it gets one, it splits the BBOX parameter on the comma, and passes it to the bboxWest,bboxSouth,bboxEast, and bboxNorth variables. It then passes these variables on to kmlOutput().

kmlOutput() iterates through the Geometry entites creating 

&lt;Placemark&gt;

 elements. For each entity, it checks to see if there is a bboxWest, and if there is it makes sure that the appropriate properties of the entity fall within the bounding box:

```
    createPlace = True
    if bboxWest != None:
      if geometry.bboxWest > bboxWest and  geometry.bboxEast < bboxEast and geometry.bboxNorth < bboxNorth and geometry.bboxSouth > bboxSouth:
        createPlace = True
      else:
        createPlace = False
    if createPlace == True:
```

If it does, then the entity is used to create a 

&lt;Placemark&gt;

, otherwise it skips on to the next entity.

### GeoPts and altitude ###
Latitude and Longitude values are stored separately as GeoPts for each geometry. This was done to demonstrate usage of GeoPt, which is a native App Engine type for coordinate data. However, GeoPt does not store altitude. This isn't a problem for 2D data, such as is used in the Maps API, but it is a problem for 3D data, such as data represented by KML. So, GeoStore stores the list of altitudes separately.

Output
Currently, GeoDataStore provides two output formats, JSON and KML. Here's an example of the JSON output:
```
{operation: 'get', status: 'success', result:{geometry:{records:[{key: 'ag5nZW9zZXJ2LW1tYXJrc3IOCxIIR2VvbWV0cnkYGQw', name: 'The Silk Road Turkish Mediterranean Restaurant', type: 'point', description: '8-10 Arcade Road', timeStamp: '2008-03-29', coordinates: [{lat: 50.808741, lng: -0.540634}], altitudes: [0.0]},
```

As you can see, each action returns a JSON object that identifies the operation and the status of that operation ('success') and has a result object. Currently, the result object has only one child object, geometry, but it is built with the possibility that other types of objects might be included. The geometry object has one child, records, an array of geometry entities, with the following name/value pairs: key (uniquely identifies the entity in the datastore), name, type (point, line, poly), description, timeStamp (when it was added to the datastore), coordinates (an array of lat and lng values) and altitudes, an array of altitude values. If the operation fails, it returns an error message.

The KML document looks very different:


Each entity is a Placemark element with a Point, LineString, or Polygon in it. The name and description properties map exactly to the name and description elements. GeoDataStore creates the appropriate Geometry elements for each of the entities.

### Front end ###

### Retrieving data ###

As described earlier, the backend provides both JSON and KML output format. Since JSON is native JavaScript data format, that output is optimal for reading into the JS frontend. For loading it, we can take advantage of the API function GDownloadURL. GDownloadURL is a wrapper for the XMLHttpRequest that's used to request a file (not necessarily XML) from the server where the HTML page resides. The first parameter to GDownloadURL is the path to your file (in this case, the URL that invokes the Python script), and the second parameter to GDownloadURL is the function that's called when the file is returned to the JavaScript.

```
var url = url_base + 'request?operation=get&output=json'
GDownloadUrl(url, function(data, responseCode) { me.handleDataResponse_(me, data, responseCode); });
```

In the callback function, you first use eval() to store the JSON output into a local variable. Then, after verifying that the status of the operation was 'success',' iterate through the records array and call ```
createGeometry_``` on each record entry.  That function is shown below:

```
geoserver.adminPanel.prototype.handleDataResponse_ = function(me, data, responseCode) {
  if (responseCode == 200) {
    var json_data = eval('(' + data + ')');
    if (json_data.status != 'success') return;
    switch (json_data.operation) {
      case 'get':
        var geometries = json_data.result.geometries;
        var bounds = new GLatLngBounds();
        for (var i = 0; i < geometries.records.length; i++) {
          var record = geometries.records[i];
          var geometry = me.createGeometry_(record);
        }
    }
  }
};
```


### Modifying data ###

The user interface allows the user to modify the geometry loaded in, provided that it was created by them originally. To make changes in the data store, we once again use GDownloadUrl. To tell the server what type of operation we're doing, we specify it in the URL that gets sent as the first argument to GDownloadUrl ("operation=add" vs. "operation="edit"). Then we transform the geometry's JSON data into form-encoded parameters ("&name=Bla&description=BlaBla"), and send the data as the third argument to GDownloadUrl. This tells the API to turn the request into a POST (more secure than a GET), and to send that data as the body of the POST. That line of code is shown below:

`GDownloadUrl(url, this.handleDataResponse_, url_params.join('&'));`


### Creating geometry ###

The GeoDataStore interface displays three types of geometry: markers (single points), polylines (arrays of coordinates), and polygons (the same as polylines, but filled). Thankfully, there are corresponding objects in the Maps API for each of those types of geometry: GMarker, GPolyline, and GPolygon, respectively. There are two situations where we must create geometry - when the map loads in the initial data, and when the user uses the EditControl and clicks the map to specify they'd like a new geometry in that location. To avoid code redundancy, we handle both of those situations with a generic createGeometry_function. That function begins by creating either the Maps API object (one of the three above), then creates the corresponding div in the sidebar, and finally assigns various event listeners to the new geometry object. The beginning of that function is shown below:_

```
geoserver.adminPanel.prototype.createGeometry_ = function(data, is_editable) {
  var me = this;
  if (data.type == 'point') {
    var geometry = new GMarker(new GLatLng(data.coordinates[0].lat, data.coordinates[0].lng), {draggable: true, icon: me.icons_.unchanged});
  } else if (data.type == 'line' || data.type == 'poly') {
    var latlngs = [];
    for (var i = 0; i < data.coordinates.length; i++) {
      latlngs.push(new GLatLng(data.coordinates[i].lat, data.coordinates[i].lng));
    }
    var geometry = (data.type == 'line') ? new GPolyline(latlngs, '#0000ff', 0,
    0.1) : new
    GPolygon(latlngs, '#0000ff', 0, 0.1, '#0000ff', 0.01);
    }
  // ... 
```