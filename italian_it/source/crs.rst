
.. _crs:

Supporto alle proiezioni 
========================

.. index:: sistemi di riferimento delle coordinate

Sistemi di Riferimento delle Coordinate
---------------------------------------

I sistemi di riferimento (CRS) sono incapsulati dalla classe :class:`QgsCoordinateReferenceSystem`.
Istanze di questa classe possono essere create in modi diversi:

* specificare un CRS tramite il suo ID::

    # PostGIS SRID 4326 denota il sistema WGS84
    crs = QgsCoordinateReferenceSystem(4326, QgsCoordinateReferenceSystem.PostgisCrsId)

  QGIS usa tre ID differenti per ogni sistema di riferimento:

  * :const:`PostgisCrsId` - ID usati dai database PostGIS.
  * :const:`InternalCrsId` - ID usati dal database di QGIS.
  * :const:`EpsgCrsId` - ID assegnati dall'organizzazione EPSG.

  Se non specificato in un secondo parametro, PostGIS SRID è usato di default.

* specificare un CRS con well-known text (WKT)::

    wkt = 'GEOGCS["WGS84", DATUM["WGS84", SPHEROID["WGS84", 6378137.0, 298.257223563]],\
           PRIMEM["Greenwich", 0.0], UNIT["degree",0.017453292519943295],\
           AXIS["Longitude",EAST], AXIS["Latitude",NORTH]]'
    crs = QgsCoordinateReferenceSystem(wkt)

* creare un CRS non valido e poi usare :func:`create*` per inizializzarlo. Nell'esempio seguente, viene usata la stringa Proj4 per inizializzare la proiezione::

    crs = QgsCoordinateReferenceSystem()
    crs.createFromProj4("+proj=longlat +ellps=WGS84 +datum=WGS84 +no_defs")

Per verificare che la creazione sia avvenuta con successo, :func:`isValid` deve restituire :const:`True`.

Si noti che per inizializzare un CRS, QGIS deve controllare i valori appropriati nel suo database interno :file:`srs.db`.
Per cui ne caso si stia creando un'applicazione standalone, è necessario impostare il percorso con :func:`QgsApplication.setPrefixPath`, altrimenti non si riuscirà ad accedere al database. Tale necessità non c'è se si usa la console python da QGIS o se si sta sviluppando un plugin.

Ottenere informazioni su un sistema di riferimento::

  print "QGIS CRS ID:", crs.srsid()
  print "PostGIS SRID:", crs.srid()
  print "EPSG ID:", crs.epsg()
  print "Description:", crs.description()
  print "Projection Acronym:", crs.projectionAcronym()
  print "Ellipsoid Acronym:", crs.ellipsoidAcronym()
  print "Proj4 String:", crs.proj4String()
  # controlla se il sistema di coordinate è geografico o proiettato
  print "Is geographic:", crs.geographicFlag()
  # controlla il tipo di unità di mappa (valori dell'enumerazione QGis::units)
  print "Map units:", crs.mapUnits()

.. index:: proiezioni

Proiezioni
----------

E' possibile operare delle trasformazioni tra sistemi di riferimento utilizzando la classe :class:`QgsCoordinateTransform`.
La maniera più semplice consiste nel creare un sistema di riferimento sorgente ed uno di destinazione e costruire l'istanza :class:`QgsCoordinateTransform` con essi. Quindi, si chiama la funzione :func:`transform` per operare la trasformazione. 
Di default la trasformazione procede dal sistema sorgente a quello di destinazione, ma è possibile operare in senso contrario::

  crsSrc = QgsCoordinateReferenceSystem(4326)    # WGS 84
  crsDest = QgsCoordinateReferenceSystem(32633)  # WGS 84 / UTM zone 33N
  xform = QgsCoordinateTransform(crsSrc, crsDest)

  # sorgente -> destinazione
  pt1 = xform.transform(QgsPoint(18,5))
  print "Transformed point:", pt1

  # destinazione -> sorgente
  pt2 = xform.transform(pt1, QgsCoordinateTransform.ReverseTransform)
  print "Transformed back:", pt2
