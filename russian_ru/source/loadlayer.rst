
.. loadlayer:

Загрузка слоёв
==============

Давайте загрузим несколько слоёв с данными. В QGIS слои делятся на векторные
и растровые. Кроме того, существуют пользовательские типы слоёв, но их
обсуждение выходит за рамки этой книги.

.. index::
  pair: векторные слои; загрузка

Векторные слои
--------------

Чтобы загрузить векторный слой нужно указать идентификатор источника данных
имя слоя и название провайдера::

  layer = QgsVectorLayer(data_source, layer_name, provider_name)
  if not layer.isValid():
    print "Layer failed to load!"

Идентификатор источника данных это строка, специфичная для каждого провайдера
векторных данных. Имя слоя используется в виджете списка слоёв.
Необходимо проверять успешно ли завершилась загрузка слоя или нет. В случае
каких-либо ошибок возвращается неправильный объект.

Ниже показано как получить доступ к различным источникам данных используя
провайдеры векторных данных:

.. index::
  pair: загрузка; слои OGR

* библиотека OGR (shape-файлы и другие форматы) --- идентификатором источника
  данных является путь к файлу::

    vlayer = QgsVectorLayer("/path/to/shapefile/file.shp", "layer_name_you_like", "ogr")

.. index::
  pair: загрузка; слои PostGIS

* база данных PostGIS --- идентификатором источника данных выступает строка
  с информацией, необходимой для установки соединения с базой данных
  PostgreSQL. Используйте класс :class:`QgsDataSourceURI` для формирования
  такой строки. Помните, что QGIS должна быть скомпилированна с поддержкой
  PostgreSQL, иначе этот провайдер будет недоступен.
  ::

    uri = QgsDataSourceURI()
    # задаем имя хоста, порт, название базы данных, имя пользователя и пароль
    uri.setConnection("localhost", "5432", "dbname", "johny", "xxx")
    # задается схема, название таблицы, поле геометрии и, опционально, подмножество (условие WHERE)
    uri.setDataSource("public", "roads", "the_geom", "cityid = 2643")

    vlayer = QgsVectorLayer(uri.uri(), "layer_name_you_like", "postgres")

.. index::
  pair: загрузка; текстовые файлы с разделителями

* CSV или другие текстовые файлы с разделителями --- для открытия файла
  с полями "x" для координаты X и "y" для координаты Y и использующего в
  качестве разделителя запятую, можно использовать такой код::

    uri = "/some/path/file.csv?delimiter=%s&xField=%s&yField=%s" % (";", "x", "y")
    vlayer = QgsVectorLayer(uri, "layer_name_you_like", "delimitedtext")

  Примечание: начиная с QGIS 1.7 строка вызова провайдера формируется в
  виде URL, поэтому путь должен начинаться с *file://*. Кроме того, допускается
  использование геометрии в формате WKT (well known text) вместо полей
  с координатами x и y, и допускатся указание желаемой системы координат.
  Например::

    uri = "file:///some/path/file.csv?delimiter=%s&crs=epsg:4723&wktField=%s" % (";", "shape")

.. index::
  pair: загрузка; файлы GPX

* GPX файлы --- провайдер данных "gpx" позволяет читать треки, маршруты
  и путевые точки из файлов gpx. При открытии файла необходимо указать
  его тип (track/route/waypoint) в качестве части url::

    uri = "path/to/gpx/file.gpx?type=track"
    vlayer = QgsVectorLayer(uri, "layer_name_you_like", "gpx")

.. index::
  pair: загрузка; слои SpatiaLite

* база данных SpatiaLite --- поддерживается начиная с QGIS v1.1. Как и в
  случае с базами PostGIS, для генерирования идентификатора источника
  данных можно использовать :class:`QgsDataSourceURI`::

    uri = QgsDataSourceURI()
    uri.setDatabase('/home/martin/test-2.3.sqlite')
    uri.setDataSource('','Towns', 'Geometry')

    vlayer = QgsVectorLayer(uri.uri(), 'Towns', 'spatialite')

.. index::
  pair: загрузка; геометрия MySQL

* WKB-геометрия из базы MySQL, доступ выполняется при помощи OGR --- в качестве
  идентификатора источника даных выступает строка подключения к таблице::

    uri = "MySQL:dbname,host=localhost,port=3306,user=root,password=xxx|layername=my_table"
    vlayer = QgsVectorLayer( uri, "my_table", "ogr" )

.. index::
  pair: растровые слои; загрузка

Растровые слои
--------------

Для работы с растровыми данными используется библиотека GDAL. Она поддерживает
множество различных форматов. В случае проблем при открытии файлов проверьте
поддерживает ли ваша версия GDAL этот формат (поддержка некоторых форматов
по умолчанию не доступна). Для загрузки растра из файла необходимо указать
его имя и имя файла::

  fileName = "/path/to/raster/file.tif"
  fileInfo = QFileInfo(fileName)
  baseName = fileInfo.baseName()
  rlayer = QgsRasterLayer(fileName, baseName)
  if not rlayer.isValid():
    print "Layer failed to load!"

.. index::
  pair: загрузка; растр WMS

Или же можно загрузить растровый слой с сервера WMS. Однако, в настоящее
время в API не предусмотрена возможность получить доступ к ответу на запрос
GetCapabilities --- необходимо знать названия нужных слоёв::

  url = 'http://wms.jpl.nasa.gov/wms.cgi'
  layers = [ 'global_mosaic' ]
  styles = [ 'pseudo' ]
  format = 'image/jpeg'
  crs = 'EPSG:4326'
  rlayer = QgsRasterLayer(0, url, 'some_layer_name', 'wms', layers, styles, format, crs)
  if not rlayer.isValid():
    print "Layer failed to load!"

.. index:: список слоёв карты

Список слоёв карты
------------------

Если вы хотите использовать открытые слои при отрисовке карты --- не забудьте
добавить их к списку слоёв карты. Список слоёв карты станет их владельцем,
а получить доступ к ним можно будет из любой части приложения по уникальному
идентификатору. При удалении слоя из списка слоёв карты, происходит его
уничтожение.

.. index:: список слоёв карты; добавление слоя

Добавление слоя в список::

  QgsMapLayerRegistry.instance().addMapLayer(layer)

При выходе слои уничтожаются автоматически, но если необходимо удалить слой
явно используйте::

  QgsMapLayerRegistry.instance().removeMapLayer(layer_id)


**TODO:**
   More about map layer registry?
