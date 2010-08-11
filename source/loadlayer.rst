
.. loadlayer:

Loading Layers
==============

Let's open some layers with data. QGIS recognizes vector and raster layers. Additionally, custom layer types are available, but we are not going to discuss them here.


Vector Layers
-------------

To load a vector layer, specify layer's data source identifier, name for the layer and provider's name::

  layer = QgsVectorLayer(data_source, layer_name, provider_name)
  if not layer.isValid():
    print "Layer failed to load!"

The data source identifier is a string and it is specific to each vector data provider. Layer's name is used in the layer list widget.
It is important to check whether the layer has been loaded successfully. If it was not, an invalid layer instance is returned.

The following list shows how to access various data sources using vector data providers:

* OGR library (shapefiles and many other file formats) - data source is the path to the file::

    vlayer = QgsVectorLayer("/path/to/shapefile/file.shp", "layer_name_you_like", "ogr")

* PostGIS database - data source is a string with all information needed to create a connection to PostgreSQL database. :class:`QgsDataSourceURI` class can generate this string for you. 
  Note that QGIS has to be compiled with Postgres support, otherwise this provider isn't available.
  ::

    uri = QgsDataSourceURI()
    # set host name, port, database name, username and password
    uri.setConnection("localhost", "5432", "dbname", "johny", "xxx")
    # set database schema, table name, geometry column and optionaly subset (WHERE clause)
    uri.setDataSource("public", "roads", "the_geom", "cityid = 2643")

    vlayer = QgsVectorLayer(uri.uri(), "layer_name_you_like", "postgres")

* CSV or other delimited text files - to open a file with a semicolon as a delimiter, with field "x" for x-coordinate and field "y" with y-coordinate you would use something like this::

    uri = "/some/path/file.csv?delimiter=%s&xField=%s&yField=%s" % (";", "x", "y")
    vlayer = QgsVectorLayer(uri, "layer_name_you_like", "delimitedtext")

* GPX files - the "gpx" data provider reads tracks, routes and waypoints from gpx files. To open a file, the type (track/route/waypoint) needs to be specified as part of the url::

    uri = "path/to/gpx/file.gpx?type=track"
    vlayer = QgsVectorLayer(uri, "layer_name_you_like", "gpx")

* SpatiaLite database - supported from QGIS v1.1. Similarly to PostGIS databases, :class:`QgsDataSourceURI` can be used for generation of data source identifier::

    uri = QgsDataSourceURI()
    uri.setDatabase('/home/martin/test-2.3.sqlite')
    uri.setDataSource('','Towns', 'Geometry')

    vlayer = QgsVectorLayer(uri.uri(), 'Towns', 'spatialite')


Raster Layers
-------------

For accessing raster files, GDAL library is used. It supports a wide range of file formats. In case you have troubles with opening some files, check whether
your GDAL has support for the particular format (not all formats are available by default). To load a raster from a file, specify its file name and base name::

  fileName = "/path/to/raster/file.tif"
  fileInfo = QFileInfo(fileName)
  baseName = fileInfo.baseName()
  rlayer = QgsRasterLayer(fileName, baseName)
  if not rlayer.isValid():
    print "Layer failed to load!"


Alternatively you can load a raster layer from WMS server. However currently it's not possible to access GetCapabilities response from API - you have to know what layers you want::

  url = 'http://wms.jpl.nasa.gov/wms.cgi'
  layers = [ 'global_mosaic' ]
  styles = [ 'pseudo' ]
  format = 'image/jpeg'
  crs = 'EPSG:4326'
  rlayer = QgsRasterLayer(0, url, 'some layer name', 'wms', layers, styles, format, crs)
  if not rlayer.isValid():
    print "Layer failed to load!"


Map Layer Registry
------------------

If you would like to use the opened layers for rendering, do not forget to add them to map layer registry. The map layer registry takes ownership of layers
and they can be later accessed from any part of the application by their unique ID. When the layer is removed from map layer registry, it gets deleted, too.

Adding a layer to the registry::

  QgsMapLayerRegistry.instance().addMapLayer(layer)

Layers are destroyed automatically on exit, however if you want to delete the layer explicitly, use::

  QgsMapLayerRegistry.instance().removeMapLayer(layer_id)


**TODO:**
   More about map layer registry?
