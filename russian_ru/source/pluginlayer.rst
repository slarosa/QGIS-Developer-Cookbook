.. index:: слои расширений

.. _pluginlayer:

Использование слоёв расширений
==============================

Если расширение использует собственные методы для отрисовки слоёв карты,
наиболее простой способ реализации --- создание нового типа слоя на основе
QgsPluginLayer.

**TODO:**
   Check correctness and elaborate on good use cases for QgsPluginLayer, ...

.. index:: слои расширений; наследование QgsPluginLayer

Наследование QgsPluginLayer
---------------------------

Ниже показан пример минимальной реализации QgsPluginLayer. Это фрагмент
`Watermark example plugin <http://github.com/sourcepole/qgis-watermark-plugin>`_::

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

При необходимости можно добавить методы для чтения и записи информации в
файл проекта::

    def readXml(self, node):

    def writeXml(self, node, doc):


Для загрузки проекта, содержащего такой слой, требуется наличие класса::

  class WatermarkPluginLayerType(QgsPluginLayerType):

    def __init__(self):
      QgsPluginLayerType.__init__(self, WatermarkPluginLayer.LAYER_TYPE)

    def createLayer(self):
      return WatermarkPluginLayer()

Кроме того, можно добавить код для отображения дополнительной информации
в окне свойств слоя::

    def showLayerProperties(self, layer):
