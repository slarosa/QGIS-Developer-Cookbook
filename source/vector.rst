
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
	Spatial filter. If empty rect is given (:obj:`QgsRectangle()`), all features are fetched. Default: empty rect
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
    intersect = index.intersects(QgsRectangle(22.5, 15.3, 23.1, 17.2))





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



Appearance (Symbology) of Vector Layers
---------------------------------------

When a vector layer is being rendered, the appearance of the data is given by **renderer** and **symbols** associated with the layer.
Symbols are classes which take care of drawing of visual representation of features, while renderers determine what symbol will be used for a particular feature.

In QGIS v1,4 a new vector rendering stack has been introduced in order to overcome the limitations of the original implementation. We refer to it as
new symbology or symbology-ng (new generation), the original rendering stack is also called old symbology. Each vector layer uses either new symbology
or old symbology, but never both at once or niether of them. It's not a global setting for all layers, so some layers might use new symbology while
others still use old symbology. In QGIS options the user can specify what symbology should be used by default when layers are loaded.
The old symbology will be kept in further QGIS v1.x releases for compatibility and we plan
to remove it in QGIS v2.0.

How to find out which implementation is currently in use::

  if layer.isUsingRendererV2():
    # new symbology - subclass of QgsFeatureRendererV2 class
    rendererV2 = layer.rendererV2()
  else:
    # old symbology - subclass of QgsRenderer class
    renderer = layer.renderer()


Note: if you plan to support also earlier versions (i.e. QGIS < 1.4), you should first check whether the :func:`isUsingRendererV2` method exists
-- if not, only old symbology is available::

  if not hasattr(layer, 'isUsingRendererV2'):
    print "You have an old version of QGIS"

We are going to focus primarily on new symbology because it has better capabilities are more options for customization.


New Symbology
^^^^^^^^^^^^^

Now that we have a reference to a renderer from the previous section, let us explore it a bit::

  print "Type:", rendererV2.type()

There are several known renderer types available in QGIS core library:

=================  =======================================  ===================================================================
Type               Class                                    Description
=================  =======================================  ===================================================================
singleSymbol       :class:`QgsSingleSymbolRendererV2`       Renders all features with the same symbol
categorizedSymbol  :class:`QgsCategorizedSymbolRendererV2`  Renders features using a different symbol for each category
graduatedSymbol    :class:`QgsGraduatedSymbolRendererV2`    Renders features using a different symbol for each range of values
=================  =======================================  ===================================================================

There might be also some custom renderer types, so never make an assumption there are just these types.
You can query :class:`QgsRendererV2Registry` singleton to find out currently available renderers.

It is possible to obtain a dump of a renderer contents in text form - can be useful for debugging::

  print rendererV2.dump()


Single Symbol Renderer
......................

You can get the symbol used for rendering by calling :func:`symbol` method and change it with :func:`setSymbol` method
(note for C++ devs: the renderer takes ownership of the symbol.)

Categorized Symbol Renderer
...........................

You can query and set attribute name which is used for classification: use :func:`classAttribute` and :func:`setClassAttribute` methods.

To get a list of categories::

  for cat in rendererV2.categories():
    print "%s: %s :: %s" % (cat.value().toString(), cat.label(), str(cat.symbol()))

Where :func:`value` is the value used for discrimination between categories, :func:`label` is a text
used for category description and :func:`symbol` method returns assigned symbol.

The renderer usually stores also original symbol and color ramp which were used for the classification:
:func:`sourceColorRamp` and :func:`sourceSymbol` methods.

Graduated Symbol Renderer
.........................

This renderer is very similar to the categorized symbol renderer described above, but instead of one attribute value per class
it works with ranges of values and thus can be used only with numerical attributes.

To find out more about ranges used in the renderer::

  for ran in rendererV2.ranges():
    print "%f - %f: %s %s" % (ran.lowerValue(), ran.upperValue(), ran.label(), str(ran.symbol()))

You can again use :func:`classAttribute` to find out classification attribute name, :func:`sourceSymbol` and :func:`sourceColorRamp` methods.
Additionally there is :func:`mode` method which determines how the ranges were created: using equal intervals, quantiles or some other method.


Working with Symbols
....................

For representation of symbols, there is :class:`QgsSymbolV2` base class with three derived classes:

 * :class:`QgsMarkerSymbolV2` - for point features
 * :class:`QgsLineSymbolV2` - for line features
 * :class:`QgsFillSymbolV2` - for polygon features

**Every symbol consists of one or more symbol layers** (classes derived from :class:`QgsSymbolLayerV2`).
The symbol layers do the actual rendering, the symbol class itself serves only as a container for the symbol layers.

Having an instance of a symbol (e.g. from a renderer), it is possible to explore it: :func:`type` method says whether it is a marker, line or fill symbol.
There is a :func:`dump` method which returns a brief description of the symbol. To get a list of symbol layers::

  for i in xrange(symbol.symbolLayerCount()):
    lyr = symbol.symbolLayer(i)
    print "%d: %s" % (i, lyr.layerType())

To find out symbol's color use :func:`color` method and :func:`setColor` to change its color.
With marker symbols additionally you can query for the symbol size and rotation with :func:`size` and :func:`angle` methods,
for line symbols there is :func:`width` method returning line width.

Size and width are in millimeters by default, angles are in degrees.

Working with Symbol Layers
..........................

As said before, symbol layers (subclasses of :class:`QgsSymbolLayerV2`) determine the appearance of the features.
There are several basic symbol layer classes for general use. It is possible to implement new symbol layer types and thus arbitrarily customize how features will be rendered.
The :func:`layerType` method uniquely identifies the symbol layer class --- the basic and default ones are SimpleMarker, SimpleLine and SimpleFill symbol layers types.
:class:`QgsSymbolLayerV2Registry` class manages a database of all available symbol layer types.

To access symbol layer data, use its :func:`properties` method that returns a key-value dictionary of properties which determine the appearance.
Each symbol layer type has a specific set of properties that it uses.
Additionally, there are generic methods :func:`color`, :func:`size`, :func:`angle`, :func:`width` with their setter counterparts.
Of course size and angle is available only for marker symbol layers and width for line symbol layers.


Creating Custom Symbol Layer Types
..................................

Imagine you would like to customize the way how the data gets rendered. You can create your own symbol layer class
that will draw the features exactly as you wish. Here is an example of a marker that draws red circles with specified radius::

  class FooSymbolLayer(QgsMarkerSymbolLayerV2):
 
    def __init__(self, radius=4.0):
      QgsMarkerSymbolLayerV2.__init__(self)
      self.radius = radius
      self.color = QColor(255,0,0)
 
    def layerType(self):
      return "FooMarker"
 
    def properties(self):
      return { "radius" : str(self.radius) }
 
    def startRender(self, context):
      pass
 
    def stopRender(self, context):
      pass
 
    def renderPoint(self, point, context):
      # Rendering depends on whether the symbol is selected (Qgis >= 1.5)
      color = context.selectionColor() if context.selected() else self.color
      p = context.renderContext().painter()
      p.setPen(color)
      p.drawEllipse(point, self.radius, self.radius)
 
    def clone(self):
      return FooSymbolLayer(self.radius)


The :func:`layerType` method determines the name of the symbol layer, it has to be unique among all symbol layers.
Properties are used for persistence of attributes. :func:`clone` method must return a copy of the symbol layer with all attributes being exactly the same.
Finally there are rendering methods: :func:`startRender` is called before rendering first feature, :func:`stopRender` when rendering is done.
And :func:`renderPoint` method which does the rendering. The coordinates of the point(s) are already transformed to the output coordinates.

For polylines and polygons the only difference would be in the rendering method: you would use :func:`renderPolyline` which receives a list of lines,
resp. :func:`renderPolygon` which receives list of points on outer ring as a first parameter and a list of inner rings (or None) as a second parameter.

Usually it is convenient to add a GUI for setting attributes of the symbol layer type to allow users to customize the appearance:
in case of our example above we can let user set circle radius. The following code implements such widget::

  class FooSymbolLayerWidget(QgsSymbolLayerV2Widget):
    def __init__(self, parent=None):
      QgsSymbolLayerV2Widget.__init__(self, parent)
 
      self.layer = None
 
      # setup a simple UI
      self.label = QLabel("Radius:")
      self.spinRadius = QDoubleSpinBox()
      self.hbox = QHBoxLayout()
      self.hbox.addWidget(self.label)
      self.hbox.addWidget(self.spinRadius)
      self.setLayout(self.hbox)
      self.connect( self.spinRadius, SIGNAL("valueChanged(double)"), self.radiusChanged)
 
    def setSymbolLayer(self, layer):
      if layer.layerType() != "FooMarker":
        return
      self.layer = layer
      self.spinRadius.setValue(layer.radius)
    
    def symbolLayer(self):
      return self.layer
 
    def radiusChanged(self, value):
      self.layer.radius = value
      self.emit(SIGNAL("changed()"))

This widget can be embedded into the symbol properties dialog. When the symbol layer type is selected in symbol properties dialog,
it creates an instance of the symbol layer and an instance of the symbol layer widget. Then it calls :func:`setSymbolLayer` method
to assign the symbol layer to the widget. In that method the widget should update the UI to reflect the attributes of the symbol layer.
:func:`symbolLayer` function is used to retrieve the symbol layer again by the properties dialog to use it for the symbol.

On every change of attributes, the widget should emit :func:`changed()` signal to let the properties dialog update the symbol preview.

Now we are missing only the final glue: to make QGIS aware of these new classes. This is done by adding the symbol layer to registry.
It is possible to use the symbol layer also without adding it to the registry, but some functionality will not work:
e.g. loading of project files with the custom symbol layers or inability to edit the layer's attributes in GUI.

We will have to create metadata for the symbol layer::

  class FooSymbolLayerMetadata(QgsSymbolLayerV2AbstractMetadata):
 
    def __init__(self):
      QgsSymbolLayerV2AbstractMetadata.__init__(self, "FooMarker", QgsSymbolV2.Marker)
 
    def createSymbolLayer(self, props):
      radius = float(props[QString("radius")]) if QString("radius") in props else 4.0
      return FooSymbolLayer(radius)
 
    def createSymbolLayerWidget(self):
      return FooSymbolLayerWidget()
 
  QgsSymbolLayerV2Registry.instance().addSymbolLayerType( FooSymbolLayerMetadata() )

You should pass layer type (the same as returned by the layer) and symbol type (marker/line/fill) to the constructor of parent class.
:func:`createSymbolLayer` takes care of creating an instance of symbol layer with attributes specified in the `props` dictionary.
(Beware, the keys are QString instances, not "str" objects).
And there is :func:`createSymbolLayerWidget` method which returns settings widget for this symbol layer type.

The last step is to add this symbol layer to the registry --- and we are done.


Creating Custom Renderers
.........................

It might be useful to create a new renderer implementation if you would like to customize the rules how to select symbols for rendering of features.
Some use cases where you would want to do it: symbol is determined from a combination of fields, size of symbols changes depending on current scale etc.

The following code shows a simple custom renderer that creates two marker symbols and chooses randomly one of them for every feature::

  import random
 
  class RandomRenderer(QgsFeatureRendererV2):
    def __init__(self, syms=None):
      QgsFeatureRendererV2.__init__(self, "RandomRenderer")
      self.syms = syms if syms else [ QgsSymbolV2.defaultSymbol(QGis.Point), QgsSymbolV2.defaultSymbol(QGis.Point) ]
  
    def symbolForFeature(self, feature):
      return random.choice(self.syms)
 
    def startRender(self, context, vlayer):
      for s in self.syms:
        s.startRender(context)
 
    def stopRender(self, context):
      for s in self.syms:
        s.stopRender(context)
 
    def usedAttributes(self):
      return []
 
    def clone(self):
      return RandomRenderer(self.syms)

The constructor of parent :class:`QgsFeatureRendererV2` class needs renderer name (has to be unique among renderers).
:func:`symbolForFeature` method is the one that decides what symbol will be used for a particular feature.
:func:`startRender` and :func:`stopRender` take care of initialization/finalization of symbol rendering.
:func:`usedAttributes` method can return a list of field names that renderer expects to be present.
Finally :func:`clone` function should return a copy of the renderer.

Like with symbol layers, it is possible to attach a GUI for configuration of the renderer.
It has to be derived from :class:`QgsRendererV2Widget`. The following sample code creates a button that allows user to set symbol of the first symbol::

  class RandomRendererWidget(QgsRendererV2Widget):
    def __init__(self, layer, style, renderer):
      QgsRendererV2Widget.__init__(self, layer, style)
      if renderer is None or renderer.type() != "RandomRenderer":
        self.r = RandomRenderer()
      else:
        self.r = renderer
      # setup UI
      self.btn1 = QgsColorButtonV2("Color 1")
      self.btn1.setColor(self.r.syms[0].color())
      self.vbox = QVBoxLayout()
      self.vbox.addWidget(self.btn1)
      self.setLayout(self.vbox)
      self.connect(self.btn1, SIGNAL("clicked()"), self.setColor1)
 
    def setColor1(self):
      color = QColorDialog.getColor( self.r.syms[0].color(), self)
      if not color.isValid(): return
      self.r.syms[0].setColor( color );
      self.btn1.setColor(self.r.syms[0].color())
 
    def renderer(self):
      return self.r

The constructor receives instances of the active layer (:class:`QgsVectorLayer`), the global style (:class:`QgsStyleV2`) and current renderer.
If there is no renderer or the renderer has different type, it will be replaced with our new renderer, otherwise we will use the current renderer
(which has already the type we need). The widget contents should be updated to show current state of the renderer.
When the renderer dialog is accepted, widget's :func:`renderer` method is called to get the current renderer -- it will be assigned to the layer.

The last missing bit is the renderer metadata and registration in registry,
otherwise loading of layers with the renderer will not work and user will not be able to select it from the list of renderers.
Let us finish our RandomRenderer example::

  class RandomRendererMetadata(QgsRendererV2AbstractMetadata):
    def __init__(self):
      QgsRendererV2AbstractMetadata.__init__(self, "RandomRenderer", "Random renderer")
 
    def createRenderer(self, element):
      return RandomRenderer()
    def createRendererWidget(self, layer, style, renderer):
      return RandomRendererWidget(layer, style, renderer)
 
  QgsRendererV2Registry.instance().addRenderer(RandomRendererMetadata())

Similarly as with symbol layers, abstract metadata constructor awaits renderer name, name visible for users and optionally name of renderer's icon.
:func:`createRenderer` method passes :class:`QDomElement` instance that can be used to restore renderer's state from DOM tree.
:func:`createRendererWidget` method creates the configuration widget. It does not have to be present or can return `None` if the renderer does not come with GUI.

To associate an icon with the renderer you can assign it in :class:`QgsRendererV2AbstractMetadata` constructor as a third (optional) argument
-- the base class constructor in the RandomRendererMetadata __init__ function becomes::

     QgsRendererV2AbstractMetadata.__init__(self, 
         "RandomRenderer", 
         "Random renderer",
         QIcon(QPixmap("RandomRendererIcon.png", "png")) )

The icon can be associated also at any later time using :func:`setIcon` method of the metadata class.
The icon can be loaded from a file (as shown above) or can be loaded from a `Qt resource <http://qt.nokia.com/doc/4.5/resources.html>`_ (PyQt4 includes .qrc compiler for Python).

Further Topics
..............

**TODO:**
 * creating/modifying symbols
 * working with style (:class:`QgsStyleV2`)
 * working with color ramps (:class:`QgsVectorColorRampV2`)
 * rule-based renderer
 * exploring symbol layer and renderer registries



Old Symbology
^^^^^^^^^^^^^

A symbol determines color, size and other properties of the feature.
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

