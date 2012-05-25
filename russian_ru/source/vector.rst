
.. _vector:

Работа с векторными слоями
==========================

Этот раздел описывает различные действия, которые можно выполнять с
векторными слоями.

**TODO:**
   Editing, Layer vs. Data provider, ...

.. index::
  triple: векторные слои; обход; объекты


Обход объектов векторного слоя
------------------------------

Ниже показано как выполнить обход всех объектов векторного слоя. Чтобы
читать объекты слоя необходимо инициализировать процесс получения при
помощи :func:`select` а потом последовательно вызывать :func:`nextFeature`
::

  provider = vlayer.dataProvider()

  feat = QgsFeature()
  allAttrs = provider.attributeIndexes()

  # начинаем получение данных: для каждого объекта запрашиваем все атрибуты и геометрию
  provider.select(allAttrs)

  # получаем каждый объект вместе с геометрией и атрибутами
  while provider.nextFeature(feat):

    # извлекаем геометрию
    geom = feat.geometry()
    print "Feature ID %d: " % feat.id() ,

    # отображаем информацю об объекте
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

    # извлекаем атрибуты
    attrs = feat.attributeMap()

    # attrs это словарь: ключ = индекс поля, значение = QgsFeatureAttribute
    # показываем все атрибуты и их значения
    for (k,attr) in attrs.iteritems():
      print "%d: %s" % (k, attr.toString())


:func:`select` позволяет получать только необходимые данные. Функция может
принимать 4 необязательных аргумента:

1. fetchAttributes
   Список атрибутов, которые нужно получать. По умолчанию: пустой список
2. rect
   Пространственный фильтр. Если передается пустой прямоугольник (:obj:`QgsRectangle()`),
   запрашиваются все объекты. По умолчанию: пустой прямоугольник
3. fetchGeometry
   Нужно ли запрашивать геометрию. По умолчанию: :const:`True`
4. useIntersect
   При использовании пространственного фильтра определяет точность проверки
   на пересечение: будет ли использоваться точная проверка или достаточно
   проверки рамок. Это необходимо, например, при использовании функции
   определения или выбора. По умолчанию: :const:`False`

Немного примеров::

  # получение объектов с геометрией и только первыми двумя полями
  provider.select([0,1])

  # получение объектов, попадающих в заданный прямоугольник, запрашивается только геометрия
  provider.select([], QgsRectangle(23.5, -10, 24.2, -7))

  # получение объектов без геометрии, но со всеми атрибутами
  allAtt = provider.attributeIndexes()
  provider.select(allAtt, QgsRectangle(), False)

Для получения индекса поля по его имени используется функция провайдера
данных :func:`fieldNameIndex`::

  fldDesc = provider.fieldNameIndex("DESCRIPTION")
  if fldDesc == -1:
    print "Field not found!"


.. index:: векторные слои; редактирование

.. _editing:

Редактирование векторных слоёв
------------------------------

Большинство провайдеров векторных данных поддерживает редактирование. Иногда
они позволяют выполнять только некоторые операции редактирования.
Узнать список доступных действий можно при помощи функции :func:`capabilities`::

  caps = layer.dataProvider().capabilities()

При использовании любого из следующих методов редактирования слоя, изменения
будут применяться к соответствующему набору данных (файлу, базе данных и т.д)
сразу же. Если необходимо произвести временные изменения, следующий раздел
можно пропустить и перейти сразу к разделу, описывающему
:ref:`редактирование с использованием буфера изменений <editing-buffer>`.

Добавление объектов
^^^^^^^^^^^^^^^^^^^

Создайте несколько экземпляров :class:`QgsFeature` и передайте список этих
объектов в метод :func:`addFeatures` провайдера. Провайдер вернет два
значения: результат (true/false) и список добавленных объектов (из ID
устанавливаются хранилищем данных)::

  if caps & QgsVectorDataProvider.AddFeatures:
    feat = QgsFeature()
    feat.addAttribute(0,"hello")
    feat.setGeometry(QgsGeometry.fromPoint(QgsPoint(123,456)))
    (res, outFeats) = layer.dataProvider().addFeatures( [ feat ] )


Удаление объектов
^^^^^^^^^^^^^^^^^

Для удаления объектов достатовно передать список идентификаторов этих
объектов::

  if caps & QgsVectorDataProvider.DeleteFeatures:
    res = layer.dataProvider().deleteFeatures([ 5, 10 ])

Изменение объектов
^^^^^^^^^^^^^^^^^^

Можно изменять как геометрию объекта так и его атрибуты. Следующий пример
сначала изменяет значения атрибутов с индексами 0 и 1, а затем модифицирует
геометрию объекта::

  fid = 100   # ID объекта, который будет редактироваться

  if caps & QgsVectorDataProvider.ChangeAttributeValues:
    attrs = { 0 : QVariant("hello"), 1 : QVariant(123) }
    layer.dataProvider().changeAttributeValues({ fid : attrs })

  if caps & QgsVectorDataProvider.ChangeGeometries:
    geom = QgsGeometry.fromPoint(QgsPoint(111,222))
    layer.dataProvider().changeGeometryValues({ fid : geom })

Добавление и удаление полей
^^^^^^^^^^^^^^^^^^^^^^^^^^^

Чтобы добавить поля (атрибуты), необходимо создать список с описанием полей.
Для удаления необходимо предоставить список с индексами удаляемых полей.
::

  if caps & QgsVectorDataProvider.AddAttributes:
    res = layer.dataProvider().addAttributes( [ QgsField("mytext", QVariant.String), QgsField("myint", QVariant.Int) ] )

  if caps & QgsVectorDataProvider.DeleteAttributes:
    res = layer.dataProvider().deleteAttributes( [ 0 ] )


.. _editing-buffer:

Редактирование векторных слоёв с использованием буфера изменений
----------------------------------------------------------------

При редактировании векторных данных в QGIS, сначала необходимо перевести
соответствущий слой в режим редактирования, затем внести изменения и, наконец,
зафиксировать (или отменить) эти изменения. Все сделанные изменения не
применяются до тех пор, пока вы их не зафиксируете --- они хранятся в буфере
изменений слоя. Данную возможность можно использовать и программно --- это
всего лишь другой способ редактирования векторных слоёв, дополняющий прямой
доступ к провайдеру. Использовать этот функционал стоит тогда, когда
пользователю предоставляются графические инструменты редактирования, чтобы
он мог решить когда фиксировать/отменять изменения, а также мог использовать
инструменты повтора/отмены. При фиксации изменений, все имеющиеся в буфере
операции будут переданы провайдеру.

Определить находится ли слоё в режиме редактирования можно при помощи
метода :func:`isEditing` --- функции редактирования работают только при
активном режиме редактирования. Применение операций редактирования показано
ниже::

  # добавление двух новых объектов (экземпляров QgsFeature)
  layer.addFeatures([feat1,feat2])
  # удаление объекта с заданным ID
  layer.deleteFeature(fid)

  # назначение новой геометрии (экземпляр QgsGeometry) объекту
  layer.changeGeometry(fid, geometry)
  # обновление атрибута с заданным индексом (int) заданным значением (QVariant)
  layer.changeAttributeValue(fid, fieldIndex, value)

  # добавление нового поля
  layer.addAttribute(QgsField("mytext", QVariant.String))
  # удаление поля
  layer.deleteAttribute(fieldIndex)

Чтобы операции повтора/отмены работали правильно, описанные выше вызовы
должны быть помещены в пакет правок. (Если вам не нужна возможность
повтора/отмены изменений и необходимо сохранять правки немедленно, все
сводится к :ref:`редактированию через провайдер <editing>`.) Вот пример
использования возможностей отмены правок::

  layer.beginEditCommand("Feature triangulation")

  # ... вызываем методы редактирования слоя ...

  if problem_occurred:
    layer.destroyEditCommand()
    return

  # ... и еще редактируем ...

  layer.endEditCommand()


Метод :func:`beginEndCommand` создаст внутреннюю "активную" команду и будет
записывать все последующие изменения в векторном слое. При вызове :func:`endEditCommand`
эта команда будет помещена в стек отмены и пользователь получит возможность
отменить/повторить её из GUI. Если в процессе редактирования что-то пошло
не так, вызов метода :func:`destroyEditCommand` удалит команду и отменит
все изменения, сделанные с момента активации этой команды.

Для актививации режима редактирования используется метод :func:`startEditing`,
за завершение редактирования отвечают :func:`commitChanges` и :func:`rollback()`
--- однако в общем случае эти методы вам не нужны, т.к. вызываться они должны
конечным пользователем.


.. index:: пространственный индекс; использование

Использование пространственного индекса
---------------------------------------

**TODO:**
   Intro to spatial indexing

1. создание пространственного индекса --- следующий код создаёт пустой индекс::

    index = QgsSpatialIndex()

2. добавление объектов к индексу --- индекс принимает объект :class:`QgsFeature`
   и добавляет его во внутреннюю структуру данных. Объект можно создать
   вручную или использовать полученные в результате предыдущих вызовов :func:`nextFeature()` ::

      index.insertFeature(feat)

3. после заполнения пространственного индекса значениями можно переходить
   к выполнению запросов::

    # возвращает массив идентификаторов пяти, ближайших к заданной точке, объектов
    nearest = index.nearestNeighbor(QgsPoint(25.4, 12.7), 5)

    # возвращает массив идентификаторов объектов, пересекающихся с заданным прямоугольником
    intersect = index.intersects(QgsRectangle(22.5, 15.3, 23.1, 17.2))



.. index:: векторные слои; запись

Запись векторных слоёв
----------------------

Для записи векторных данных на диск служит класс :class:`QgsVectorFileWriter`.
Он позволяет создавать векторные файлы в любом, поддерживаемом OGR, формате
(shape-файлы, GeoJSON, KML и другие).

Существует два способа записать векторные данные в файл:

* из экземпляра :class:`QgsVectorLayer`::

    error = QgsVectorFileWriter.writeAsVectorFormat(layer, "my_shapes.shp", "CP1250", None, "ESRI Shapefile")
    if error == QgsVectorFileWriter.NoError:
      print "success!"

    error = QgsVectorFileWriter.writeAsVectorFormat(layer, "my_json.json", "utf-8", None, "GeoJSON")
    if error == QgsVectorFileWriter.NoError:
      print "success again!"

Третий параметр задает конечную кодировку текста. Он требуется некоторым
драйверам (в частности драйверу shape-файлов) для нормальной работы. В
случае если вы не используете международные символы, специально заботиться
о правильной кодировке не нужно. Четвертый параметр, который мы оставили
пустым, задает целевую систему координат - если передан корректный экземпляр
:class:`QgsCoordinateReferenceSystem`, слой будет трансформирован в эту
систему координат.

Узнать правильные названия драйверов можно на странице `supported formats by OGR`_
- в качестве имени драйвера используется значение колонки "Code". При необходимости
можно экспортировать только выделенные объекты, передать дополнительные
параметры драйверу или запретить сохранение атрибутов - с полным синтаксисом
можно ознакомиться в документации.

.. _supported formats by OGR: http://www.gdal.org/ogr/ogr_formats.html


* из отдельных объектов::

    # определяем поля для атрибутов объекта
    fields = { 0 : QgsField("first", QVariant.Int),
               1 : QgsField("second", QVariant.String) }

    # создаем экземпляр класса для записи векторных данных. Аргументы:
    # 1. путь к новому файлу (если такой файл уже существует, возникнет ошибка)
    # 2. кодировка атрибутивных данных
    # 3. список полей
    # 4. тип геометрии --- из перечислимого типа WKBTYPE
    # 5. система координат слоя (экземпляр QgsCoordinateReferenceSystem) --- опционально
    # 6. имя используемого драйвера
    writer = QgsVectorFileWriter("my_shapes.shp", "CP1250", fields, QGis.WKBPoint, None, "ESRI Shapefile")

    if writer.hasError() != QgsVectorFileWriter.NoError:
      print "Error when creating shapefile: ", writer.hasError()

    # добавляем объекты
    fet = QgsFeature()
    fet.setGeometry(QgsGeometry.fromPoint(QgsPoint(10,10)))
    fet.addAttribute(0, QVariant(1))
    fet.addAttribute(1, QVariant("text"))
    writer.addFeature(fet)

    # уничтожаем объект класса и сбрасываем изменения на диск (опционально)
    del writer

.. index:: memory провайдер

Memory провайдер
----------------

Memory провайдер в основном предназначен для использования разработчиками
расширений или сторонних приложений. Этот провайдер не хранит данные на
диске, что позволят разработчикам использовать его в качестве быстрого
хранилища для временных слоёв.

Провайдер поддерживает строковые и целочисленные поля, а также поля с
плавающей запятой.

Memory провайдер помимо всего прочего поддерживает и пространственное
индексирование, пространственный индекс можно создать вызовав функцию
:func:`createSpatialIndex` провайдера. После создания пространственного
индекса обход объектов в пределах небольшой области станет более быстрым
(поскольку обращение будет идти только к объектам, попадающим в заданный
прямоугольник).

Memory провайдер будет использоваться если в качестве идентификатора провайдера
при вызове конструктора :class:`QgsVectorLayer` указана строка ``"memory"``.

В конструктор также передается URI, описывающий геометрию слоя, это может
быть: ``"Point"``, ``"LineString"``, ``"Polygon"``, ``"MultiPoint"``,
``"MultiLineString"`` или ``"MultiPolygon"``.

Начиная с QGIS 1.7 URI также может содержать описание системы координат,
описание полей и включать пространственное индексирование. Используется
следующий синтаксис:

crs=definition
    Задает используемую систему координат, definition может принимать любой
    вид, совместимый с :func:`QgsCoordinateReferenceSystem.createFromString`

index=yes
    Определяет будет ли провайдер использовать пространственный индекс

field=name:type(length,precision)
    Задает атрибуты слоя. Каждый атрибут имеет имя и, опционально, тип
    (целое число, вещественное число или строка), длину и точность.
    Таких описаний может быть несколько.

Ниже показан пример URI, содержащий все описанные выше параметры::

  "Point?crs=epsg:4326&field=id:integer&field=name:string(20)&index=yes"

Следующий пример кода показывает процесс создания и наполнения memory провайдера::

  # создается слой
  vl = QgsVectorLayer("Point", "temporary_points", "memory")
  pr = vl.dataProvider()

  # добавляются поля
  pr.addAttributes( [ QgsField("name", QVariant.String),
                      QgsField("age",  QVariant.Int),
                      QgsField("size", QVariant.Double) ] )

  # добавляется объект
  fet = QgsFeature()
  fet.setGeometry( QgsGeometry.fromPoint(QgsPoint(10,10)) )
  fet.setAttributeMap( { 0 : QVariant("Johny"),
                         1 : QVariant(20),
                         2 : QVariant(0.3) } )
  pr.addFeatures( [ fet ] )

  # после добавления новых объектов обновляем охват слоя
  # т.к. изменение охвата в провайдере не распространяется на слой
  vl.updateExtents()

И, наконец, проверим что всё прошло успешно::

  # отображаем информацию о слое
  print "fields:", pr.fieldCount()
  print "features:", pr.featureCount()
  e = pr.extent()
  print "extent:", e.xMin(),e.yMin(),e.xMax(),e.yMax()

  # проходим по всем объектам
  f = QgsFeature()
  pr.select()
  while pr.nextFeature(f):
    print "F:",f.id(), f.attributeMap(), f.geometry().asPoint()

.. index:: векторные слои; символика

Внешний вид (символика) векторных слоёв
---------------------------------------

При отрисовке векторного слоя, внешний вид данных определяется **рендером**
и **символами**, ассоциированными со слоем. Символы это классы, занимающиеся
отрисовкой визуального представления объектов, а рендер опеределяет какой
символ будет использован для отдельного объекта.

В QGIS v1.4 был введен новый механизм отрисовки векторных слоев, призванный
устранить ограничения оригинальной реализации. Мы называем его новой символикой
или symbology-ng (new generation --- новое поколение), оригинальную реализацию
также называют старой символикой. Каждый векторный слой использует либо новую
либо старую символику, но нельзя использовать обе одновременно, или ни одну
из них. Это не глобальная настройка для всех слоёв, то есть некоторые слои
могут использовать новую символику, в то время как другие продолжают
использовать старую. В настройках QGIS пользователь может указать какую
символику необходимо использовать по умолчанию. Старая символика будет
сохранена в будущих выпусках QGIS 1.x для совместимости, но мы планируем
отказаться от неё в QGIS v2.0.

Узнать какая реализация используется можно так::

  if layer.isUsingRendererV2():
    # новая символика --- подкласс класса QgsFeatureRendererV2
    rendererV2 = layer.rendererV2()
  else:
    # старая символика --- подкласс класса QgsRenderer
    renderer = layer.renderer()


Примечание: если требуется поддержка старых версий (например, QGIS < 1.4),
вначале необходимо проверить существует ли метод :func:`isUsingRendererV2`
--- если его нет, доступна только старая символика::

  if not hasattr(layer, 'isUsingRendererV2'):
    print "You have an old version of QGIS"

В первую очередь и более подробно мы рассмотрим новую символику, так как
она обладает большими возможностями и предоставляет больше настроек.

.. index:: символика; новая

Новая символика
^^^^^^^^^^^^^^^

Теперь, имея ссылку на рендер из предыдущего раздела, изучим его поближе::

  print "Type:", rendererV2.type()

В библиотеке ядра QGIS реализовано несколько рендеров:

=================  =======================================  =============================================================================
Тип                Класс                                    Описание
=================  =======================================  =============================================================================
singleSymbol       :class:`QgsSingleSymbolRendererV2`       Отрисовывает все объекты одним и тем же символом
categorizedSymbol  :class:`QgsCategorizedSymbolRendererV2`  Отрисовывает объекты, используя разные символы для каждой категории
graduatedSymbol    :class:`QgsGraduatedSymbolRendererV2`    Отрисовывает объекты, используя разные символы для каждого диапазона значений
=================  =======================================  =============================================================================

Кроме того, могут быть доступны пользовательские рендеры, поэтому не стоит
предполагать, что присутствуют только вышеназванные типы. Узнать список
доступных рендеров можно обратившись к синглтону :class:`QgsRendererV2Registry`.

Существует возможность получить дамп содержимого рендера в текстовом виде,
это может быть полезно при отладке::

  print rendererV2.dump()

.. index:: рендер обычным знаком, символика; рендер обычным знаком

Рендер обычным знаком
.....................

Получить символ, используемый для отрисовки, можно вызвав метод :func:`symbol`,
а для его изменения служит метод :func:`setSymbol` (примечание для пишущих на
C++: рендер становится владельцем символа).

.. index:: рендер уникальными значениями, символика; hендер уникальными значениями

Рендер уникальными значениями
.............................

Узнать и задать поле атрибутивной таблицы, используемое для классификации
можно при помощи методов :func:`classAttribute` и :func:`setClassAttribute`
соответственно.

А так получаем список значений::

  for cat in rendererV2.categories():
    print "%s: %s :: %s" % (cat.value().toString(), cat.label(), str(cat.symbol()))

Здесь :func:`value` --- величина, используемая для разделения категорий,
:func:`label` --- описание категории, а метод :func:`symbol` возвращает
назначеный символ.

Также рендер обычно сохраняет оригинальный символ и цветовую шкалу, которые
использовались для классификации, получить их можно вызвав методы
:func:`sourceColorRamp` и :func:`sourceSymbol` соответственно.

.. index:: символика; рендер градуированным знаком, рендер градуированным знаком

Рендер градуированным знаком
............................

Этот рендер очень похож на рендер уникальными значениями, описанный выше,
но вместо одного значения атрибута для класса он оперирует диапазоном значений
и следовательно, может использоваться только с числовыми атрибутами.

Получить информацию о диапазонах, используемых рендером::

  for ran in rendererV2.ranges():
    print "%f - %f: %s %s" % (ran.lowerValue(), ran.upperValue(), ran.label(), str(ran.symbol()))

Как и в предыдущем случае, доступны методы :func:`classAttribute` для получения
имени атрибута классификации, :func:`sourceSymbol` и :func:`sourceColorRamp`
чтобы узнать оригинальный символ и цветовую шкалу. Кроме того, дополнительный
метод :func:`mode` позволяет узнать какой алгоритм использовался для создания
диапазонов: равные интервалы, квантили или что-то другое.

Если вы хотите создать свой рендер категориями, можете воспользоваться
следующим фрагментом кода в качестве отправной точки. Пример ниже создает
простое разделение объектов на два класса::

  from qgis.core import (QgsVectorLayer, QgsMapLayerRegistry,
                         QgsGraduatedSymbolRendererV2,
                         QgsSymbolV2, QgsRendererRangeV2)

  myVectorLayer = QgsVectorLayer(myVectorPath, myName, 'ogr')
  myTargetField = myStyle['target_field']
  myRangeList = []
  myOpacity = 1
  # создаем первый символ и диапазон...
  myMin = 0.0
  myMax = 50.0
  myLabel = 'Group 1'
  myColour = QtGui.QColor('#ffee00')
  mySymbol1 = QgsSymbolV2.defaultSymbol(
       myVectorLayer.geometryType())
  mySymbol.setColor(myColour)
  mySymbol.setAlpha(myOpacity)
  myRange1 = QgsRendererRangeV2(
            myMin,
            myMax,
            mySymbol1,
            myLabel)
  myRangeList.append(myRange1)
  # теперь создаем другой символ и диапазоне...
  myMin = 50.1
  myMax = 100
  myLabel = 'Group 2'
  myColour = QtGui.QColor('#00eeff')
  mySymbol2 = QgsSymbolV2.defaultSymbol(
       myVectorLayer.geometryType())
  mySymbol.setColor(myColour)
  mySymbol.setAlpha(myOpacity)
  myRange2 = QgsRendererRangeV2(
            myMin,
            myMax,
            mySymbol2
            myLabel)
  myRangeList.append(myRange2)
  myRenderer = QgsGraduatedSymbolRendererV2(
            '', myRangeList)
  myRenderer.setMode(
    QgsGraduatedSymbolRendererV2.EqualInterval)
  myRenderer.setClassAttribute(myTargetField)

  myVectorLayer.setRendererV2(myRenderer)
  QgsMapLayerRegistry.instance().addMapLayer(myVectorLayer)


.. index:: символы; работа с

Работа с символами
..................

Символы представлены базовым классом :class:`QgsSymbolV2` и тремя классами
наследниками:

 * :class:`QgsMarkerSymbolV2` - для точечных объектов
 * :class:`QgsLineSymbolV2` - для линейных объектов
 * :class:`QgsFillSymbolV2` - для полигональных объектов

**Каждый символ состоит из одного и более символьных слоёв** (классы, унаследованные от
:class:`QgsSymbolLayerV2`).
Всю работу по отрисовке выполняют слои символа, а символ служит только
контейнером для них.

Получив экземпляр символа (например, от рендера), можно заняться его изучением:
метод :func:`type` расскажет является ли этот символ маркером, линией или
заливкой. Метод :func:`dump` вернет краткое описание символа. А получить
список слоёв символа можно так::

  for i in xrange(symbol.symbolLayerCount()):
    lyr = symbol.symbolLayer(i)
    print "%d: %s" % (i, lyr.layerType())

Узнать цвет символа можно вызвав метод :func:`color`, а чтобы изменить
его --- :func:`setColor`. У символов типа маркер присутствуют дополнительные
методы :func:`size` и :func:`angle`, позволяющие узнать размер символа и угол
поворота, а у линейных символов есть метод :func:`width`, возвращающий
толщину линии.

Размер и толщина по умолчанию задаются в миллиметрах, а углы --- в градусах.

.. index:: символьные слои; работа с

Работа со слоями символа
........................

Как уже было сказано, слои символа (наследники :class:`QgsSymbolLayerV2`)
определяют внешний вид объектов. Существует несколько базовых классов
символьных слоёв. Кроме того, можно создавать новые символьные слои и таким
образом влиять на отрисовку объектов в достаточно широких пределах. Метод
:func:`layerType` однозначно идентифицирует класс символьного слоя ---
основными и доступными по умолчанию являются символьные слои
SimpleMarker, SimpleLine и SimpleFill.

Получить полный список символьных слоёв, которые можно использовать в
заданном символьном слое, можно так::

  from qgis.core import QgsSymbolLayerV2Registry
  myRegistry = QgsSymbolLayerV2Registry.instance()
  myMetadata = myRegistry.symbolLayerMetadata("SimpleFill")
  for item in myRegistry.symbolLayersForType(QgsSymbolV2.Marker):
    print item

Результат::

  EllipseMarker
  FontMarker
  SimpleMarker
  SvgMarker
  VectorField

Класс :class:`QgsSymbolLayerV2Registry` управляет базой всех доступных
символьных слоёв.

Получить доступ к данным символьного слоя можно при помощи метода :func:`properties`,
который возвращает словарь (пары ключ-значение) свойств, влияющих на внешний
вид. Символьные слои каждого типа имеют свой набор свойств. Кроме того,
существуют общие для всех типов методы :func:`color`, :func:`size`,
:func:`angle`, :func:`width` и соответсвующие им сеттеры. Следует помнить,
что размер и угол поворота доступны только для символьных слоёв типа маркер,
а толщина --- только для слоёв типа линия.

.. index:: символьные слои; создание пользовательских

Создание пользовательских символьных слоёв
..........................................

Представьте, что вам необходимо настроить процесс отрисовки своих данных.
Для этого можно создать свой собственный класс символьного слоя, который
будет рисовать объекты именно так, как вам нужно. Вот пример маркера,
рисующего красные окружности заданного радиуса::

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
      # Отрисовка зависит от того выделен символ или нет (Qgis >= 1.5)
      color = context.selectionColor() if context.selected() else self.color
      p = context.renderContext().painter()
      p.setPen(color)
      p.drawEllipse(point, self.radius, self.radius)

    def clone(self):
      return FooSymbolLayer(self.radius)


Метод :func:`layerType` определяет имя символьного слоя, которое должно
быть уникальным. Чтобы все атрибуты были неизменными, используются свойства.
Метод :func:`clone` должен возвращать копию символьного слоя с точно такими
же атрибутами. И наконец, методы отрисовки: :func:`startRender` вызывается
перед отрисовкой первого объекта, а :func:`stopRender` --- после окончания
отрисовки. За собственно отрисовку отвечает метод :func:`renderPoint`.
Координаты точки (точек) должны быть трансформирваны в выходные координаты.

Для полининий и полигонов единственное отличие будет в методе отрисовки:
необходимо использовать :func:`renderPolyline`, принимающий список линий,
или :func:`renderPolygon` в качестве первого аргумента принимающий
список точек, образующих внешнее кольцо, и список внутренних колец (или None)
вторым аргументом.

Хорошей практикой является создание интерфейса для управления атрибутами
символьного слоя, что позволяет пользователям настраивать внешний вид:
в случае нашего примера, можно предоставить пользователю возможность менять
радиус окружности. Реализовать это можно так::

  class FooSymbolLayerWidget(QgsSymbolLayerV2Widget):
    def __init__(self, parent=None):
      QgsSymbolLayerV2Widget.__init__(self, parent)

      self.layer = None

      # создаем простой интерфейс
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

Этот виджет можно встроить в диалог свойств символа. Когда символьный слой
выделяется в диалоге свойств символа, создается экземпляр символьного слоя
и экземпляр виджета символьного слоя. Затем вызывается метод :func:`setSymbolLayer`
чтобы привязать символьный слой к виджету. В этом методе виджет должен
обновить интерфейс, чтобы отразить атрибуты символьного слоя. Функция
:func:`symbolLayer` используется диалогом свойств для получения измененного
символьного слоя для дальнейшего использования.

При каждом изменении атрибутов виджет должен посылать сигнал :func:`changed()`,
чтобы диалог свойств мог обновить предпросмотр символа.

Остался последний штрих: рассказать QGIS о существовании этих новых классов.
Для этого достаточно добавить символьный слой в реестр. Конечно, можно
использовать символьный слой и не добавляя его в реестр, но тогда некоторые
возможности будут недоступны: например, загрузка проекта с пользовательскими
символьными слоями или невозможность редактировать свойства слоя.

Сначала нужно создать метаданные символьного слоя::

  class FooSymbolLayerMetadata(QgsSymbolLayerV2AbstractMetadata):

    def __init__(self):
      QgsSymbolLayerV2AbstractMetadata.__init__(self, "FooMarker", QgsSymbolV2.Marker)

    def createSymbolLayer(self, props):
      radius = float(props[QString("radius")]) if QString("radius") in props else 4.0
      return FooSymbolLayer(radius)

    def createSymbolLayerWidget(self):
      return FooSymbolLayerWidget()

  QgsSymbolLayerV2Registry.instance().addSymbolLayerType( FooSymbolLayerMetadata() )

В конструктор родительского класса необходимо передать тип слоя
(тот же, что сообщает слой) и тип символа (маркер/линия/заливка).
:func:`createSymbolLayer` создаёт экземпляр символьного слоя с атрибутами,
указаными в словаре `props`. (Будьте внимательны, ключи являются экземплярами
QString, а не объектами "str"). Метод :func:`createSymbolLayerWidget` должен
возвращать виджет настроек этого символьного слоя.

Последней конструкцией мы добавляем символьный слой в реестр --- на этом все.

.. index::
  pair: пользовательские; рендеры

Создание пользовательских рендеров
..................................

Возможность создать свой рендер может быть полезной, если требуется
изменить правила выбора символов для отрисовки объектов. Примерами таких
ситуаций могут быть: символ должен определяться на основании значений
нескольких полей, размер символа должен зависеть от текущего масштаба и т.д.

Следующий код демонстрирует простой пользовательский рендер, который
создает два маркера и случаным образом выбирает один из них при отрисовке
каждого объекта::

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

В конструктор родительского класса :class:`QgsFeatureRendererV2` необходимо
передать имя ренедера (должно быть уникальным). Метод :func:`symbolForFeature`
определяет какой символ будет использоваться для конкретного объекта.
:func:`startRender` и :func:`stopRender` выполняют инициализацию/финализацию
отрисовки символа. Метод :func:`usedAttributes` может возвращать список
имен полей, которые необходимы рендеру. И, наконец, функция :func:`clone`
должна возвращать копию рендера.

Как и в случае символьных слоёв, рендер может иметь интерфейс для настройки
параметров. Он наследуется от класса :class:`QgsRendererV2Widget`. Следующий
код создает кнопку, позволяющую пользователю изменять один из символов::

  class RandomRendererWidget(QgsRendererV2Widget):
    def __init__(self, layer, style, renderer):
      QgsRendererV2Widget.__init__(self, layer, style)
      if renderer is None or renderer.type() != "RandomRenderer":
        self.r = RandomRenderer()
      else:
        self.r = renderer
      # создание интерфейса
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

В конструктор передается экземпляры активного слоя (:class:`QgsVectorLayer`),
глобальный стиль (:class:`QgsStyleV2`) и текущий рендер. Если рендер не
задан или имеет другой тип, он будет заменен нашим рендером, в противном
случае мы будем использовать текущий рендер (который нам и нужен). Необходимо
обновить содержимое виджета, чтобы отразить текущее состояние рендера.
При закрытии диалога ренедера, вызывается метод :func:`renderer` виджета
чтобы получить текущий рендер --- он будет назначен слою.

Осталось немного: метаданные рендера и его регистрация в реестре, иначе
загрузить слои с этим рендером не получится, а пользователь не увидит его
в списке доступных рендеров. Закончим наш пример с RandomRenderer::

  class RandomRendererMetadata(QgsRendererV2AbstractMetadata):
    def __init__(self):
      QgsRendererV2AbstractMetadata.__init__(self, "RandomRenderer", "Random renderer")

    def createRenderer(self, element):
      return RandomRenderer()
    def createRendererWidget(self, layer, style, renderer):
      return RandomRendererWidget(layer, style, renderer)

  QgsRendererV2Registry.instance().addRenderer(RandomRendererMetadata())

Так же, как и в случае символьных слоёв, абстрактный конструктор метаданных
должен получить имя рендера, отображаемое имя и, по желанию, название иконки
рендера. Метод :func:`createRenderer` получает экземпляр :class:`QDomElement`,
который может использоваться для восстановления состояния рендера из дерева DOM.
Метод :func:`createRendererWidget` отвечает за создание виджета настройки.
Он может отсутствовать или возвращать `None`, если рендер не имеет интрерфейса.

Назначить иконку рендеру можно передав её в конструктор :class:`QgsRendererV2AbstractMetadata`
в качестве третьего (необязательного) аргумента --- конструктор базового
класса в функции __init__ класса RandomRendererMetadata примет вид::

     QgsRendererV2AbstractMetadata.__init__(self,
         "RandomRenderer",
         "Random renderer",
         QIcon(QPixmap("RandomRendererIcon.png", "png")) )

Иконку можно назначить и позже, воспользовавшись методом :func:`setIcon`
класса метаданных. Иконка может загружаться из файла (как показано выше)
или из `ресурсов Qt <http://qt.nokia.com/doc/4.5/resources.html>`_ (в составе
PyQt4 присутствует компилятор .qrc для Python).

Further Topics
..............

**TODO:**
 * creating/modifying symbols
 * working with style (:class:`QgsStyleV2`)
 * working with color ramps (:class:`QgsVectorColorRampV2`)
 * rule-based renderer
 * exploring symbol layer and renderer registries

.. index:: символика; старая

Старая символика
^^^^^^^^^^^^^^^^

Знак определяет цвет, размер и другие свойства объекта. Рендер, ассоциированный
со слоем, решает какой знак будет использован для определённого объекта.
Всего доступно четыре рендера:

* обычный знак (:class:`QgsSingleSymbolRenderer`) --- все объекты отображаются одним и тем же знаком.
* уникальное значение (:class:`QgsUniqueValueRenderer`) --- знак для каждого объекта выбирается на основании значения атрибута.
* градуированный знак (:class:`QgsGraduatedSymbolRenderer`) --- знак применяется к группе (классу) объектов, разбиение на классы выполняется по числовому полю
* непрерывное значение (:class:`QgsContinuousSymbolRenderer`)

Создать точечный знак можно так::

  sym = QgsSymbol(QGis.Point)
  sym.setColor(Qt.black)
  sym.setFillColor(Qt.green)
  sym.setFillStyle(Qt.SolidPattern)
  sym.setLineWidth(0.3)
  sym.setPointSize(3)
  sym.setNamedPointSymbol("hard:triangle")

Метод :func:`setNamedPointSymbol` определяет фигуру, которая будет
использоваться для знака. Существует два класса: встроенные символы
(начинается с ``hard:``) и SVG символы (начинаются с ``svg:``). Доступны
следующие встроенные символы: ``circle``, ``rectangle``, ``diamond``,
``pentagon``, ``cross``, ``cross2``, ``triangle``, ``equilateral_triangle``,
``star``, ``regular_star``, ``arrow``.

Для создания SVG знака выполните::

  sym = QgsSymbol(QGis.Point)
  sym.setNamedPointSymbol("svg:Star1.svg")
  sym.setPointSize(3)

SVG символы не поддерживают установку цвета, заливки и стиля линии.

Создание линейного знака::

  TODO

Создание площадного знака::

  TODO

Создание рендера обычным знаком::

  sr = QgsSingleSymbolRenderer(QGis.Point)
  sr.addSymbol(sym)

Назначение рендера слою::

  layer.setRenderer(sr)

Рендер уникальным значением::

  TODO

Рендер градуированным значением::

    # задаем поле классификации и желаемое количество классов
    fieldName = "My_Field"
    numberOfClasses = 5

    # получаем номер поля по его имени
    fieldIndex = layer.fieldNameIndex(fieldName)

    # создаем ренедер, которые позже будет назначен слою
    renderer = QgsGraduatedSymbolRenderer( layer.geometryType() )

    # настраиваем режим рендера, можно выбрать один из EqualInterval/Quantile/Empty
    renderer.setMode( QgsGraduatedSymbolRenderer.EqualInterval )

    # задаем классы (нижнюю и верхнюю границы и подпись для каждого класса)
    provider = layer.dataProvider()
    minimum = provider.minimumValue( fieldIndex ).toDouble()[ 0 ]
    maximum = provider.maximumValue( fieldIndex ).toDouble()[ 0 ]

    for i in range( numberOfClasses ):
        # строку форматирования надо изменить в зависимости  от типа данных (целые или с плавающей точкой)
        lower = ('%.*f' % (2, minimum + ( maximum - minimum ) / numberOfClasses * i ) )
        upper = ('%.*f' % (2, minimum + ( maximum - minimum ) / numberOfClasses * ( i + 1 ) ) )
        label = "%s - %s" % (lower, upper)
        color = QColor(255*i/numberOfClasses, 0, 255-255*i/numberOfClasses)
        sym = QgsSymbol( layer.geometryType(), lower, upper, label, color )
        renderer.addSymbol( sym )

    # устанавливаем поле классификации и назначаем рендер слою
    renderer.setClassificationField( fieldIndex )

    layer.setRenderer( renderer )

