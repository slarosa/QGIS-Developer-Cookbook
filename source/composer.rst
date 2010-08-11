
.. _composer:

Map Rendering and Printing
==========================

There are generally two approaches when input data should be rendered as a map: either do it quick way using :class:`QgsMapRenderer` or
produce more fine-tuned output by composing the map with :class:`QgsComposition` class and friends.


Simple Rendering
----------------

Render some layers using :class:`QgsMapRenderer` - create destination paint device (``QImage``, ``QPainter`` etc.), set up layer set, extent, output size and do the rendering::

  # create image
  img = QImage(QSize(800,600), QImage.Format_ARGB32_Premultiplied)

  # set image's background color
  color = QColor(255,255,255)
  img.fill(color.rgb())

  # create painter
  p = QPainter()
  p.begin(img)
  p.setRenderHint(QPainter.Antialiasing)

  render = QgsMapRender()

  # set layer set
  lst = [ layer.getLayerID() ]  # add ID of every layer
  render.setLayerSet(lst)

  # set extent
  rect = QgsRect(render.fullExtent())
  rect.scale(1.1)
  render.setExtent(rect)

  # set output size
  render.setOutputSize(img.size(), img.logicalDpiX())

  # do the rendering
  render.render(p)

  p.end()

  # save image
  img.save("render.png","png")


Output using Map Composer
-------------------------


The following piece of code renders layers from map canvas with the current extent into a PNG file. The default settings
for composition are page size A4 and resolution 300 DPI (it's possible to change them).
::

  from qgis.core import *
  from qgis.utils import iface
  from PyQt4.QtCore import *
  from PyQt4.QtGui import *

  # set up composition
  mapRenderer = iface.mapCanvas().mapRenderer()
  c = QgsComposition(mapRenderer)
  c.setPlotStyle(QgsComposition.Print)

  dpi = c.printResolution()
  dpmm = dpi / 25.4
  width = int(dpmm * c.paperWidth())
  height = int(dpmm * c.paperHeight())

  # add a map to the composition
  x, y = 0, 0
  w, h = c.paperWidth(), c.paperHeight()
  composerMap = QgsComposerMap(c, x,y,w,h)
  c.addItem(composerMap)

  # create output image and initialize it
  image = QImage(QSize(width, height), QImage.Format_ARGB32)
  image.setDotsPerMeterX(dpmm * 1000)
  image.setDotsPerMeterY(dpmm * 1000)
  image.fill(0)

  # render the composition
  imagePainter = QPainter(image)
  sourceArea = QRectF(0, 0, c.paperWidth(), c.paperHeight())
  targetArea = QRectF(0, 0, width, height)
  c.render(imagePainter, targetArea, sourceArea)
  imagePainter.end()

  image.save("out.png", "png")


**TODO:**
   Output to PDF,
   Loading/saving compositions,
   More composer items (north arrow, scale, ...)
   
