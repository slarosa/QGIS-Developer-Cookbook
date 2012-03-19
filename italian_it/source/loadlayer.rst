
.. loadlayer:

Caricare Layer
==============

Apriamo qualche layer di dati. QGIS gestisce layer vettoriali e raster. Inoltre, è possibile utilizzare layer personalizzati, ma non se ne discuterà qui.

.. index:: 
  pair: layer vettoriali; caricare

Layer vettoriali
----------------

Per caricare un layer bisogna specificare l'identificatore della fonte dati (``data_source``), il nome del layer (``layer_name``) e del fornitore (``provider_name``)::

  layer = QgsVectorLayer(data_source, layer_name, provider_name)
  if not layer.isValid():
    print "Layer failed to load!"

L'identificatore della fonte dati è una stringa specifica per ogni fornitore di dati vettoriali. Il nome del layer viene usato nella
TOC (Table Of Content) di QGIS dove è presente la lista dei layers. E' importante controllare se il layer è stato caricato correttamente.

L'elenco seguente mostra alcuni esempi di come accedere a diverse fonti di dati:

.. index:: 
  pair: caricare; layer OGR

* Libreria OGR (shapefile ed vari altri formati di file) - la fonte dati è il percorso al file::

    vlayer = QgsVectorLayer("/path/to/shapefile/file.shp", "layer_name_you_like", "ogr")

.. index:: 
  pair: caricare; layer PostGIS

* Database PostGIS - la fonte dati è una stringa contenente le informazioni necessarie a creare una connessione ad un database PostgreSQL. La classe :class:`QgsDataSourceURI` viene utilizzata per generare questa stringa. 
  Si noti che QGIS deve essere compilato con il supporto a Postgres, altrimenti questo fornitore non sarà disponibile.
  ::

    uri = QgsDataSourceURI()
    # set host name, port, database name, username and password
    uri.setConnection("localhost", "5432", "dbname", "johny", "xxx")
    # set database schema, table name, geometry column and optionaly subset (WHERE clause)
    uri.setDataSource("public", "roads", "the_geom", "cityid = 2643")

    vlayer = QgsVectorLayer(uri.uri(), "layer_name_you_like", "postgres")

.. index:: 
  pair: caricare; layer testo delimitato

* CSV o altri file a testo delimitato - per aprire un file con delimitatore ';', con un campo "x" per le coordinate X ed una cmpo "y" per le coordinate Y, si può usare::

    uri = "/some/path/file.csv?delimiter=%s&xField=%s&yField=%s" % (";", "x", "y")
    vlayer = QgsVectorLayer(uri, "layer_name_you_like", "delimitedtext")

  .. note:: a partire dalla versione 1.7 di QGIS, la stringa del fornitore è strutturata come un URL, con prefisso del percorso *file://*. E' possibile usare geometrie formattate (well known text) e specificare il sistema di riferimento delle coordinate. 
  
  Ad esempio::

    uri = "file:///some/path/file.csv?delimiter=%s&crs=epsg:4723&wktField=%s" % (";", "shape")

.. index::
  pair: caricare; file GPX

* File GPX - il fornitore "gpx" legge track, route e waypoint da un file gpx. Per aprire un file, il tipo (track/route/waypoint) deve essere specificato come parte dell'url attraverso l'elemento ``type=``::

    uri = "path/to/gpx/file.gpx?type=track"
    vlayer = QgsVectorLayer(uri, "layer_name_you_like", "gpx")

.. index::
  pair: caricare; layer SpatiaLite

* Database SpatiaLite - a partire dalla versione 1.1 di QGIS. Alla stessa maniera di PostGIS, la classe :class:`QgsDataSourceURI` può essere usata per generare l'identificatore della fonte dati::

    uri = QgsDataSourceURI()
    uri.setDatabase('/home/martin/test-2.3.sqlite')
    uri.setDataSource('','Towns', 'Geometry')

    vlayer = QgsVectorLayer(uri.uri(), 'Towns', 'spatialite')

.. index::
  pair: caricare; geometrie MySQL

* Geometrie WKB MySQL, attraverso OGR - la fonte dati è la stringa di connessione alla tabella::
    
    uri = "MySQL:dbname,host=localhost,port=3306,user=root,password=xxx|layername=my_table"
    vlayer = QgsVectorLayer( uri, "my_table", "ogr" )

.. index:: 
  pair: layer raster; caricare
  
Layer Raster
------------

Il caricamento di file raster richiede l'uso della libreria GDAL. GDAL supporta un ampio range di formati file. In caso di problemi nel caricamento di un file, controllare che GDAL abbia il supporto per il formato specifico. Per caricare un raster da un file, specificare il nome del file ed il nome con cui si vuole identificare il layer nella TOC di QGIS (``baseName``)::

  fileName = "/path/to/raster/file.tif"
  fileInfo = QFileInfo(fileName)
  baseName = fileInfo.baseName()
  rlayer = QgsRasterLayer(fileName, baseName)
  if not rlayer.isValid():
    print "Layer failed to load!"

Nell'esempio, ``baseName`` è impostato al nome del file raster senza l'estensione. E' possibile assegnare un nome a proprio piacimento sostituendo la stringa::

  rlayer = QGSRasterLayer(fileName, baseName)

con::

  rlayer = QGSRasterLayer(fileName, "nome del layer nella toc")

.. index::
  pair: caricare; raster WMS

E' possibile anche caricare un raster da un server WMS. Attualmente non è possibile accedere al *GetCapabilities*, per cui bisogna conoscere il layer che si vuole caricare::

  url = 'http://wms.jpl.nasa.gov/wms.cgi'
  layers = [ 'global_mosaic' ]
  styles = [ 'pseudo' ]
  format = 'image/jpeg'
  crs = 'EPSG:4326'
  rlayer = QgsRasterLayer(0, url, 'some layer name', 'wms', layers, styles, format, crs)
  if not rlayer.isValid():
    print "Layer failed to load!"

.. index:: registro layer di mappa

Registro dei layer di mappa
---------------------------
.. da controllare

Per visualizzare un layer aperto, non dimenticare di aggiungerlo al registro dei layer di mappa. Il registro diventa "proprietario" dei layer, che possono poi essere utilizzati in ogni parte dell'applicazione tramite il loro ID univoco. Rimuovere un layer dal registro equivale a cancellarlo.

.. index:: registro layer mappa; aggiungere un layer

Per aggiungere un layer::

  QgsMapLayerRegistry.instance().addMapLayer(layer)

Per eliminare un layer (si noti che i layer sono eliminati automaticamente all'uscita)::

  QgsMapLayerRegistry.instance().removeMapLayer(layer_id)

**TODO:**
   More about map layer registry?
