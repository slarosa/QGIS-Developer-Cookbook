
.. _crs:

Работа с проекциями
===================

.. index:: системы координат

Системы координат
-----------------

Системы координат (Coordinate reference system, CRS) инкапсулируются классом
:class:`QgsCoordinateReferenceSystem`. Создать экземпляр этого класса можно
одним из способов:

* задать CRS по её ID::

    # WGS84 имеет PostGIS SRID 4326
    crs = QgsCoordinateReferenceSystem(4326, QgsCoordinateReferenceSystem.PostgisCrsId)

  QGIS использует три разных идентификатора для каждой системы координат:

  * :const:`PostgisCrsId` --- идентификатор, используемый в базах PostGIS.
  * :const:`InternalCrsId` --- внутренний идентификатор, используемый в базе QGISe.
  * :const:`EpsgCrsId` --- идентификатор, назначенный EPSG

  По умолчанию используется PostGIS SRID, если иное не определено вторым параметром.

* задать CRS по её представлению в формате WKT (well-known text)::

    wkt = 'GEOGCS["WGS84", DATUM["WGS84", SPHEROID["WGS84", 6378137.0, 298.257223563]],\
           PRIMEM["Greenwich", 0.0], UNIT["degree",0.017453292519943295],\
           AXIS["Longitude",EAST], AXIS["Latitude",NORTH]]'
    crs = QgsCoordinateReferenceSystem(wkt)

* создать недействительную CRS, а затем использовать одну из функций :func:`create*`
  для её инициализации. В примере ниже для инициализации проекции используется
  строка в формате Proj4::

    crs = QgsCoordinateReferenceSystem()
    crs.createFromProj4("+proj=longlat +ellps=WGS84 +datum=WGS84 +no_defs")

Желательно проверить успешность создания (т.е выполнить поиск в базе данных)
системы координат: :func:`isValid` должна вернуть :const:`True`.

Заметим, что для инициализации систем координат QGIS требуется выполнить
поиск необходимых значений в своей внутренней базе данных :file:`srs.db`.
Поэтому, если создаётся самостоятельное приложение, необходимо правильно
настроить пути при помощи :func:`QgsApplication.setPrefixPath`, иначе не
удастся найти базу данных. В случае создания расширения или при работе
в консоли Python QGIS беспокоиться не о чем: всё уже настроено.

Получение информации о системе координат::

  print "QGIS CRS ID:", crs.srsid()
  print "PostGIS SRID:", crs.srid()
  print "EPSG ID:", crs.epsg()
  print "Description:", crs.description()
  print "Projection Acronym:", crs.projectionAcronym()
  print "Ellipsoid Acronym:", crs.ellipsoidAcronym()
  print "Proj4 String:", crs.proj4String()
  # определяем тип системы координат: географическая или спроецированная
  print "Is geographic:", crs.geographicFlag()
  # единицы карты, используемые в этой CRS (определены в перечислимом типе QGis::units)
  print "Map units:", crs.mapUnits()

.. index:: проекции

Проекции
--------

Для преобразования между разными системами координат используется класс
:class:`QgsCoordinateTransform`. Наиболее простой способ использования ---
создать объекты для исходной и целевой систем координат и инициализировать
ими экземпляр класса :class:`QgsCoordinateTransform`. После чего можно
выполнять преобразование, вызывая функцию :func:`transform`. По умолчанию
выполняется прямое преобразование, но можно осуществлять и обратное::

  crsSrc = QgsCoordinateReferenceSystem(4326)    # WGS 84
  crsDest = QgsCoordinateReferenceSystem(32633)  # WGS 84 / UTM zone 33N
  xform = QgsCoordinateTransform(crsSrc, crsDest)

  # прямое преобразование: src -> dest
  pt1 = xform.transform(QgsPoint(18,5))
  print "Transformed point:", pt1

  # обратное преобразование: dest -> src
  pt2 = xform.transform(pt1, QgsCoordinateTransform.ReverseTransform)
  print "Transformed back:", pt2

