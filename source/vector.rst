
.. _vector:

Using Vector Layers
===================

This section summarizes various actions that can be done with vector layers.

**TODO:**
   Editing, Layer vs. Data provider, ...

Iterating over Vector Layer
---------------------------

Below is an example how to go through the features of the layer. To read features from layer, initialize the retieval with :func:`select` and then use :func:`nextFeature` calls::

  provider = vlayer.dataProvider()

  feat = QgsFeature()
  allAttrs = provider.attributeIndexes()

  # start data retreival: fetch geometry and all attributes for each feature
  provider.select(allAttrs)

  # retreive every feature with its geometry and attributes
  while provider.nextFeature(feat):

    # fetch geometry
    geom = feat.geometry()
    print "Feature ID %d: " % feat.id() ,

    # show some information about the feature
    if geom.vectorType() == QGis.Point:
      x = geom.asPoint()
      print "Point: " + str(x)
    elif geom.vectorType() == QGis.Line:
      x = geom.asPolyline()
      print "Line: %d points" % len(x)
    elif geom.vectorType() == QGis.Polygon:
      x = geom.asPolygon()
      numPts = 0
      for ring in x:
	numPts += len(ring)
      print "Polygon: %d rings with %d points" % (len(x), numPts)
    else:
      print "Unknown"

    # fetch map of attributes
    attrs = feat.attributeMap()
    
    # attrs is a dictionary: key = field index, value = QgsFeatureAttribute
    # show all attributes and their values
    for (k,attr) in attrs.iteritems():
      print "%d: %s" % (k, attr.toString())


:func:`select` gives you flexibility in what data will be fetched. It can get 4 arguments, all of them are optional:

1. fetchAttributes
	List of attributes which should be fetched. Default: empty list
2. rect
	Spatial filter. If empty rect is given (:obj:`QgsRect()`), all features are fetched. Default: empty rect
3. fetchGeometry
	Whether geometry of the feature should be fetched. Default: :const:`True`
4. useIntersect
	When using spatial filter, this argument says whether accurate test for intersection should be done or whether test on bounding box suffices.
	This is needed e.g. for feature identification or selection. Default: :const:`False`

Some examples::

  # fetch features with geometry and only first two fields
  provider.select([0,1])

  # fetch features with geometry which are in specified rect, attributes won't be retreived
  provider.select([], QgsRectangle(23.5, -10, 24.2, -7))

  # fetch features without geometry, with all attributes
  allAtt = provider.attributeIndexes()
  provider.select(allAtt, QgsRectangle(), False)

To obtain field index from its name, use provider's :func:`fieldNameIndex` function::

  fldDesc = provider.fieldNameIndex("DESCRIPTION")
  if fldDesc == -1:
    print "Field not found!"



Using Spatial Index
-------------------

**TODO:**
   Intro to spatial indexing

1. create spatial index - the following code creates an empty index::

    index = QgsSpatialIndex()

2. add features to index - index takes :class:`QgsFeature` object and adds it to the internal data structure.
   You can create the object manually or use one from previous call to provider's :func:`nextFeature()` ::

      index.insertFeature(feat)

3. once spatial index is filled with some values, you can do some queries::

    # returns array of feature IDs of five nearest features
    nearest = index.nearestNeighbor(QgsPoint(25.4, 12.7), 5)

    # returns array of IDs of features which intersect the rectangle
    intersect = index.intersects(QgsRect(22.5, 15.3, 23.1, 17.2))





Writing Shapefiles
------------------

You can write shapefiles using :class:`QgsVectorFileWriter` class. Besides shapefiles, it supports any kind of vector file that OGR supports.

There are two possibilities how to export a shapefile:

* from an instance of :class:`QgsVectorLayer`::

    error = QgsVectorFileWriter.writeAsShapefile(layer, "my_shapes.shp", "CP1250")

    if error == QgsVectorFileWriter.NoError:
      print "success!"

* directly from features::

    # define fields for feature attributes
    fields = { 0 : QgsField("first", QVariant.Int),
               1 : QgsField("second", QVariant.String) }

    # create an instance of vector file writer, it will create the shapefile. Arguments:
    # 1. path to new shapefile (will fail if exists already)
    # 2. encoding of the attributes
    # 3. field map
    # 4. geometry type - from WKBTYPE enum
    # 5. layer's spatial reference (instance of QgsCoordinateReferenceSystem) - optional
    writer = QgsVectorFileWriter("my_shapes.shp", "CP1250", fields, QGis.WKBPoint, None)

    if writer.hasError() != QgsVectorFileWriter.NoError:
      print "Error when creating shapefile: ", writer.hasError()

    # add some features
    fet = QgsFeature()
    fet.setGeometry(QgsGeometry.fromPoint(QgsPoint(10,10)))
    fet.addAttribute(0, QVariant(1))
    fet.addAttribute(1, QVariant("text")) 
    writer.addFeature(fet)

    # delete the writer to flush features to disk (optional)
    del writer




Memory Provider
---------------

Memory provider is intended to be used mainly by plugin or 3rd party app developers.
It does not store data on disk, allowing developers to use it as a fast backend for some temporary layers (until now one had to use e.g. OGR provider).
You can use it by passing ``"memory"`` provider string to vector layer constructor.

Following URIs are allowed: "Point" / "LineString" / "Polygon" / "MultiPoint" / "MultiLineString" / "MultiPolygon" - for different types of data stored in the layer.

The provider supports string, int and double fields.

Following example should be self-explaining::

  # create layer
  vl = QgsVectorLayer("Point", "temporary_points", "memory")
  pr = vl.dataProvider()

  # add fields 
  pr.addAttributes( [ QgsField("name", QVariant.String), 
                      QgsField("age",  QVariant.Int), 
                      QgsField("size", QVariant.Double) ] )

  # add a feature
  fet = QgsFeature()
  fet.setGeometry( QgsGeometry.fromPoint(QgsPoint(10,10)) )
  fet.setAttributeMap( { 0 : QVariant("Johny"), 
                         1 : QVariant(20), 
                         2 : QVariant(0.3) } )
  pr.addFeatures( [ fet ] )

  # update layer's extent when new features have been added
  # because change of extent in provider is not propagated to the layer
  vl.updateExtents()

Finally, let's check whether everything went well::

  # show some stats
  print "fields:", pr.fieldCount()
  print "features:", pr.featureCount()
  e = pr.extent()
  print "extent:", e.xMin(),e.yMin(),e.xMax(),e.yMax()

  # iterate over features
  f = QgsFeature()
  pr.select()
  while pr.nextFeature(f):
    print "F:",f.id(), f.attributeMap(), f.geometry().asPoint()

Memory provider also supports spatial indexing. This means you can call provider's :func:`createSpatialIndex` function.
Once the spatial index is created (using :class:`QgsSpatialIndex` class), you will be able to iterate over features within smaller regions faster
(since it's not necessary to traverse all the features, only those in specified rectangle). 


Controlling Symbology of Vector Layers
--------------------------------------

When a vector layer is being rendered, the appearance of the data is given by one or more symbols. A symbol
determines color, size and other properties of the feature.
Renderer associated with the layer decides what symbol will be used for particular feature. There are
four available renderers:

* single symbol renderer (:class:`QgsSingleSymbolRenderer`) --- all features are rendererd with the same symbol.
* unique value renderer (:class:`QgsUniqueValueRenderer`) --- symbol for each feature is choosen from attribute value.
* graduated symbol renderer (:class:`QgsGraduatedSymbolRenderer`)
* continuous color renderer (:class:`QgsContinuousSymbolRenderer`)

How to create a point symbol::

  sym = QgsSymbol(QGis.Point)
  sym.setColor(Qt.black)
  sym.setFillColor(Qt.green)
  sym.setFillStyle(Qt.SolidPattern)
  sym.setLineWidth(0.3)
  sym.setPointSize(3)
  sym.setNamedPointSymbol("hard:triangle")

The :func:`setNamedPointSymbol` method determines the shape of the symbol. There are two classes:
hardcoded symbols (prefixed ``hard:``) and SVG symbols (prefixed ``svg:``). The following hardcoded
symbols are available: ``circle``, ``rectangle``, ``diamond``, ``pentagon``, ``cross``, ``cross2``, ``triangle``,
``equilateral_triangle``, ``star``, ``regular_star``, ``arrow``.

How to create an SVG symbol::

  sym = QgsSymbol(QGis.Point)
  sym.setNamedPointSymbol("svg:Star1.svg")
  sym.setPointSize(3)

SVG symbols do not support setting colors, fill and line styles.

How to create a line symbol::

  TODO

How to create a fill symbol::

  TODO

Create a single symbol renderer::

  sr = QgsSingleSymbolRenderer(QGis.Point)
  sr.addSymbol(sym)

Assign the renderer to a layer::

  layer.setRenderer(sr)

Create unique value renderer::

  TODO

Create graduated symbol renderer::

  TODO

