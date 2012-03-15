.. index:: layer plugin

.. _pluginlayer:

Usare i layer plugin
====================

Il miglior modo per implementare un plugin che usa i propri metodi per la rappresentazione dei layer di mappa consiste nell'utilizzare QgsPluginLayer.

**TODO:**
   Check correctness and elaborate on good use cases for QgsPluginLayer, ...

.. index:: layer plugin; derivare classi da QgsPluginLayer

Derivare classi da QgsPluginLayer
---------------------------------

Di seguito un esempio di un'implementazione basica di QgsPluginLayer, derivato da `Watermark example plugin <http://github.com/sourcepole/qgis-watermark-plugin>`_::

  class WatermarkPluginLayer(QgsPluginLayer):

    LAYER_TYPE="watermark"

    def __init__(self):
      QgsPluginLayer.__init__(self, WatermarkPluginLayer.LAYER_TYPE, "Watermark plugin layer")
      self.setValid(True)

    def draw(self, rendererContext):
      image = QImage("myimage.png")
      painter = rendererContext.painter()
      painter.save()
      painter.drawImage(10, 10, image)
      painter.restore()
      return True

Possono essere aggiunti metodi per leggere e scrivere informazioni specifiche nel file di progetto::

    def readXml(self, node):

    def writeXml(self, node, doc):

E' necessaro usare una classe "factory"::

  class WatermarkPluginLayerType(QgsPluginLayerType):

    def __init__(self):
      QgsPluginLayerType.__init__(self, WatermarkPluginLayer.LAYER_TYPE)

    def createLayer(self):
      return WatermarkPluginLayer()

Si può aggiungere codice per visualizzare informazioni personalizzate nelle proprietà del layer::

    def showLayerProperties(self, layer):
