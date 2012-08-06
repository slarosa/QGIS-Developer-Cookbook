
.. _vector:

Utilizzare layer Vettoriali
===========================

Questa sezione riassume le varie operazioni possibili con i layer vettoriali.


.. index:: 
  triple: layer vettoriali; iterare; elementi

Iterazione su layer vettoriali
------------------------------

Di seguito un esempio su come accedere ed iterare tra gli elementi di un layer vettoriale. Per leggere gli elementi da un layer, usare :func:`select` e poi :func:`nextFeature`::

  provider = vlayer.dataProvider()

  feat = QgsFeature()
  allAttrs = provider.attributeIndexes()

  # avvio recupero dati: acquisisce geometria ed attributi di ogni elemento
  provider.select(allAttrs)

  # recupera ogni elemento, con la sua geometria ed attributi
  while provider.nextFeature(feat):

    # recupera geometria
    geom = feat.geometry()
    print "Feature ID %d: " % feat.id() ,

    # mostra informazioni sull'elemento
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

    # recupera la mappa degli attributi
    attrs = feat.attributeMap()
    
    # attrs è un dizionario: key = indice del campo, value = QgsFeatureAttribute
    # mostra tutti gli attributi ed i loro valori
    for (k,attr) in attrs.iteritems():
      print "%d: %s" % (k, attr.toString())


La funzione :func:`select` offre la possibilità di scegliere quali dati acquisire; ha quattro argomenti opzionali:

1. fetchAttributes
	Elenca gli attributi da acquisire. Default: lista vuota
2. rect
	Filtro spaziale. Se vuoto (:obj:`QgsRectangle()`), saranno acquisiti tutti gli elementi. Default: vuoto
3. fetchGeometry
	Definisce se acquisire la geometria di un elemento. Default: :const:`True`
4. useIntersect
	Quando si usa il filtro spaziale, questo argomento definisce se effettuare un test accurato delle intersezioni or se effettuare un test sul solo bounding box.
	E' necessario, ad esempio, per l'identificazione e selezione di elementi. Default: :const:`False`

Alcuni esempi::

  # recupera gli elementi con geometria e solo i primi due campi
  provider.select([0,1])

  # recupera gli elementi con geometria che si trovano all'interno di un rettangolo, gli attributi non sono recuperati
  provider.select([], QgsRectangle(23.5, -10, 24.2, -7))

  # recupera gli elementi senza geometria con tutti gli attributi
  allAtt = provider.attributeIndexes()
  provider.select(allAtt, QgsRectangle(), False)

Per ottenere l'indice del campo a partire dal suo nome, usare la funzione del fornitore :func:`fieldNameIndex`::

  fldDesc = provider.fieldNameIndex("DESCRIPTION")
  if fldDesc == -1:
    print "Campo non trovato!"


.. index:: layer vettoriali; modifica

.. _modifica:

Modifica di layer vettoriali
----------------------------

La maggior parte dei fornitori di dati vettoriali supportano la modifica dei dati; alcune volte supportano giusto un sottoinsieme delle possibili funzionalità di modifica.
Per ottenere l'insieme di funzionalità supportate da uno specifico fornitore, usare la funzione :func:`capabilities`::

    caps = layer.dataProvider().capabilities()

Utilizzando i metodi successivi per la modifica dei layer vettoriali, i cambiamenti sono direttamente salvati (nei file o database corrispondenti). La sezione :ref:`Modificare layer vettoriali con buffer di modifica <editing-buffer>` spiega come effettuare dei cambiamenti temporanei.

Aggiungere elementi
^^^^^^^^^^^^^^^^^^^

Per aggiungere degli elementi, creare delle istanze di :class:`QgsFeature` e passarle al metodo del fornitore :func:`addFeatures`. Verranno restituiti due valori: il risultato (true/false) ed una lista degli elementi aggiunti (il loro ID è impostato dal *data store*)::

    if caps & QgsVectorDataProvider.AddFeatures:
    feat = QgsFeature()
    feat.addAttribute(0,"hello")
    feat.setGeometry(QgsGeometry.fromPoint(QgsPoint(123,456)))
    (res, outFeats) = layer.dataProvider().addFeatures( [ feat ] )


Eliminare elementi
^^^^^^^^^^^^^^^^^^

Per eliminare degli elementi basta fornire la lista dei loro ID::

    if caps & QgsVectorDataProvider.DeleteFeatures:
    res = layer.dataProvider().deleteFeatures([ 5, 10 ])

Modificare elementi
^^^^^^^^^^^^^^^^^^^

E' possibile modificare la geometria o gli attributi di un elemento. Nell'esempio seguente vengono modificati i valori degli attributi con indice 0 e 1 e la geometria::
   
   fid = 100   # ID degli elementi da modificare
      
   if caps & QgsVectorDataProvider.ChangeAttributeValues:
    attrs = { 0 : QVariant("hello"), 1 : QVariant(123) }
    layer.dataProvider().changeAttributeValues({ fid : attrs })
    
   if caps & QgsVectorDataProvider.ChangeGeometries:
    geom = QgsGeometry.fromPoint(QgsPoint(111,222))
    layer.dataProvider().changeGeometryValues({ fid : geom })

Aggiungere e rimuovere campi
^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Per aggiungere dei campi (attributi), è necessario specificare una lista delle definizioni di campo. Per cancellare dei campi, basta fornire una lista degli ID di campo::

    if caps & QgsVectorDataProvider.AddAttributes:
      res = layer.dataProvider().addAttributes( [ QgsField("mytext", QVariant.String), QgsField("myint", QVariant.Int) ] )
    
    if caps & QgsVectorDataProvider.DeleteAttributes:
      res = layer.dataProvider().deleteAttributes( [ 0 ] )


.. _editing-buffer:

Modificare layer vettoriali con buffer di modifica
--------------------------------------------------

Quando si modificano dei vettori in QGIS, come prima operazione bisogna attivare la sessione di modifica, quindi si apportano le modifiche ed infine si confermano le modifiche chiudendo la sessione di modifica. I cambiamenti non sono scritti finchè non si chiude la sessione di modifica - vengono memorizzati temporaneamente in un buffer di modifica. E' possibile utilizzare questa funzionalità anche per la programmazione - si tratta di usare un metodo per la modifica dei layer vettoriali a complemento dell'utilizzo diretto del fornitore. Utilizzare questa opzione per strumenti di modifica di layer vettoriali con GUI, in modo da permettere all'utente di scegliere se salvare o meno i cambiamenti e di usare gli strumenti Annulla/Ripristina.
Quando si salvano i cambiamenti, le modifiche del buffer vengono salvate nel fornitore.

Per capire se un layer è in modalità di modifica, usare :func:`isEditing` - la funzione di modifica funziona solo se la modalità di modifica è attiva. Esempi di utilizzo delle funzioni di modifica::

  # aggiungere due elementi (istanze di QgsFeature)
  layer.addFeatures([feat1,feat2])
  # eliminare un elementi con specifico ID
  layer.deleteFeature(fid)

  # impostare una nuova geometria (istanza di QgsGeometry)
  layer.changeGeometry(fid, geometry)
  # aggiornare un attributo con specifico ID (QVariant)
  layer.changeAttributeValue(fid, fieldIndex, value)

  # aggiungere un nuovo campo
  layer.addAttribute(QgsField("miotesto", QVariant.String))
  # rimuovere un campo
  layer.deleteAttribute(fieldIndex)

Per attivare una sessione di modifica usare il metodo :func:`startEditing`; :func:`commitChanges` e :func:`rollBack()` permettono di chiudere la sessione di modifica; questi ultimi due metodi, comunque, non dovrebbero essere usati, lasciando all'utente la possibilità di utilizzare tale funzionalità.

Per il corretto funzionamento dei comando Annulla/Ripristina, le chiamate di cui sopra devono essere concatenate all'interno 
di comandi di annullamento delle operazione di modifica.
(Se non si vuole utilizzare il comando Annulla/Riprisitna e quindi si vogliono salvare immediatamente tutti le modifiche sul layer, 
allora si avrà la possibilità di utilizzare più facilmente la :ref:`modifica con il fornitori di dati <modifica>`.) Come si utilizza la funzionalità di Annulla/Ripristina::

  layer.beginEditCommand("Feature triangulation")
  
  # ... chiamata ai metodi di modifica di un layer vettoriale ...
  
  if problem_occurred:
    layer.destroyEditCommand()
    return
  
  # ... altre operazioni di modifica sul layer vettoriale ...
  
  layer.endEditCommand()

La funzione :func:`beginEndCommand` crea un comando "attivo" e registra tutti i cambiamenti che avvengono nel layer vettoriale. 
Mentre, con la chiamata alla funzione :func:`endEditCommand` il comando viene inserito nello stack di annullamento e l'utente sarà in grado di eseguire un Annulla/Ripristina dalla GUI (interfaccia grafica). Nel caso in cui qualcosa è andato storto mentre si fanno le modifiche, il metodo :func:`destroyEditCommand` rimuoverà il comando ed annullerà tutte le modifiche fatte, mentre il comando è attivo.

Per attivare una sessione di modifica usare il metodo :func:`startEditing`; :func:`commitChanges` e :func:`rollBack()` permettono di chiudere la sessione di modifica; questi ultimi due metodi, comunque, non dovrebbero essere usati, lasciando all'utente la possibilità di utilizzare tale funzionalità.

.. index:: indice spaziale; utilizzare

Utilizzare l'indice spaziale
----------------------------

**TODO:**
   Intro to spatial indexing

1. Creare un indice spaziale - il codice seguente permette di creare un indice vuoto::

    index = QgsSpatialIndex()

2. Aggiungere elementi ad un indice - l'indice prende l'oggetto :class:`QgsFeature` e lo aggiunge alla struttura interna dei dati.
   L'oggetto può essere creato manualmente o può essere usato un oggetto dalla chiamata precedente alla funzione del fornitore:func:`nextFeature()`::

      index.insertFeature(feat)

3. Una volta riempito l'indice di valori, è possibile operare delle query::

    # restituisce un array degli ID dei cinque elementi più vicini
    nearest = index.nearestNeighbor(QgsPoint(25.4, 12.7), 5)

    # restituisce un array degli ID degli elementi che intersecano il rettangolo
    intersect = index.intersects(QgsRectangle(22.5, 15.3, 23.1, 17.2))
 

.. index:: layer vettoriale; scrivere

Scrivere un layer vettoriale
----------------------------

E' possibile scrivere un file vettoriale usando la classe :class:`QgsVectorFileWriter`, che supporta tutti i formati di OGR.

Ci sono due possibilità per esportare un layer vettoriale:

* da un'istanza di :class:`QgsVectorLayer`::

    error = QgsVectorFileWriter.writeAsVectorFormat(layer, "my_shapes.shp", "CP1250", None, "ESRI Shapefile")

    if error == QgsVectorFileWriter.NoError:
      print "success!"

    error = QgsVectorFileWriter.writeAsVectorFormat(layer, "my_json.json", "utf-8", None, "GeoJSON")
    if error == QgsVectorFileWriter.NoError:
      print "success again!"

Il terzo parametro specifica la codifica dell'output. Solo alcuni driver necessitano di specificare tale parametro (non è il caso degli shapefile): ad ogni modo, se non si usano caratteri internazionali, non c'è necessità di preoccuparsene. Il quarto parametro (impostato a None nell'esempio) permette di impostate il sistema di riferimento per le coordinate dell'output, se viene passata una istanza valida di :class:`QgsCoordinateReferenceSystem`.

Per i nomi dei driver, consultare `i formati supportati da OGR`_ - usare il valore del campo "Code" come nome del driver. E' possibile definire se esportare i soli elementi selezionati, passare altre opzioni specifiche del driver, se creare o meno gli attributi - si guardi la documentazione per la sintassi completa.

.. _i formati supportati da OGR: http://www.gdal.org/ogr/ogr_formats.html


* direttamente dagli elementi::

    # defisce i campi per gli attributi
    fields = { 0 : QgsField("first", QVariant.Int),
               1 : QgsField("second", QVariant.String) }

    # istanza dello "scrittore" del file vettoriale. Argomenti:
    # 1. percorso al nuovo file (fallisce se il percorso esiste già)
    # 2. codifica degli attributi
    # 3. mappa dei campi
    # 4. tipo di geometia - enumerazione WKBTYPE
    # 5. Sistema di riferimento (istanza di QgsCoordinateReferenceSystem) - opzionale
    # 6. Nome del driver per il file di output
    writer = QgsVectorFileWriter("my_shapes.shp", "CP1250", fields, QGis.WKBPoint, None, "ESRI Shapefile")

    if writer.hasError() != QgsVectorFileWriter.NoError:
      print "Error when creating shapefile: ", writer.hasError()

    # aggiunge alcuni elementi
    fet = QgsFeature()
    fet.setGeometry(QgsGeometry.fromPoint(QgsPoint(10,10)))
    fet.addAttribute(0, QVariant(1))
    fet.addAttribute(1, QVariant("text")) 
    writer.addFeature(fet)

    # cancella lo "scrittore" (opzionale)
    del writer

.. index:: fornitore di memoria

Fornitore di memoria
--------------------

Il fornitore di memoria è dedicato agli sviluppatori di plugin o di applicazioni di terze parti.
Non memorizzando i dati sul disco, permette agli sviluppatori di disporre di un backend veloce per i layer temporanei.

Il fornitore supporta campi string, int e double.

Il fornitore di memoria, inoltre, supporta l'indicizzazione tramite la funzione :func:`createSpatialIndex`. Creato l'indice spaziale, è possibile iterare tra gli elementi in modo più veloce.

Un fornitore di memoria si crea passando ``"memory"`` come stringa del fornitore al costruttore :class:`QgsVectorLayer`.

Il costruttore, inoltre, prende in input un URI per definire il tipo di geometria del layer: ``"Point"``, ``"LineString"``, ``"Polygon"``, ``"MultiPoint"``, ``"MultiLineString"``, o ``"MultiPolygon"``.

A partire dalla versione 1.7 di QGIS l'URI può anche specificare il sistema di riferimento, i campi e l'indicizzazione del fornitore di memoria.
La sintassi è la seguente:

crs=definition
    Specifica il sistema di riferimento, in una delle forme accettate da :func:`QgsCoordinateReferenceSystem.createFromString`

index=yes
    Specifica che il fornitre utilizzerà l'indice spaziale

field=name:type(length,precision)
    Specifica un attributo del layer. L'attributo ha un nome e opzionalmente tipo (integer, double, o string), lunghezza e precisione.
    Ci possono essere più definizione di campo.

Segue un esempio di URI contenente tutte le opzioni::

  "Point?crs=epsg:4326&field=id:integer&field=name:string(20)&index=yes"

Il codice seguente mostra come creare e popolare un fornitore di memoria::

  # crea il layer
  vl = QgsVectorLayer("Point", "temporary_points", "memory")
  pr = vl.dataProvider()

  # aggiunge i campi 
  pr.addAttributes( [ QgsField("name", QVariant.String), 
                      QgsField("age",  QVariant.Int), 
                      QgsField("size", QVariant.Double) ] )

  # aggiunge un elemento
  fet = QgsFeature()
  fet.setGeometry( QgsGeometry.fromPoint(QgsPoint(10,10)) )
  fet.setAttributeMap( { 0 : QVariant("Johny"), 
                         1 : QVariant(20), 
                         2 : QVariant(0.3) } )
  pr.addFeatures( [ fet ] )

  # aggiorna l'estensione del layer all'aggiunta di un nuovo elemento
  # in quanto i cambiamenti nel provider non sono propagati al layer
  vl.updateExtents()

Per controllare l'esattezza dell'operazione::

  # mostra alcune statistiche
  print "fields:", pr.fieldCount()
  print "features:", pr.featureCount()
  e = pr.extent()
  print "extent:", e.xMin(),e.yMin(),e.xMax(),e.yMax()

  # itera tra gli elementi
  f = QgsFeature()
  pr.select()
  while pr.nextFeature(f):
    print "F:",f.id(), f.attributeMap(), f.geometry().asPoint()

.. index:: layer vettoriali; simbologia

Simbologia per i layer vettoriali
---------------------------------

Le modalità di visualizzazione dei dati di un layer vettoriale sono determinate da
un **visualizzatore** e da **simboli**. I simboli sono classi che di occupano dei 
fornire la rappresentazione visuale di elementi, mentre i visualizzatori determinano
quale simbolo utilizzare per un dato elemento.

A partire da  QGIS v1.4 è stato introdotto un nuovo stack di visualizzazione al fine
di risolvere i limiti dell'implementazione originaria. Il nuovo stack è noto come
nuova simbologia o simbologia-ng (nuova generazione), il vecchio stack come vecchia simbologia.
Ogni layer vettoriale usa o la vecchia o la nuova simbologia, ma non entrambe i nessuna delle due.
La modalità di visualizzazione non è un'impostazione globale, in modo da poter usare stack diversi per
layer diversi. In QGIS l'utente può definire la simbologia di default da usare quando vengono 
caricati dei layer. La vecchia simbologia sarà mantenuta per le versioni 1.x di QGIS, ma sarà
rimossa a partire dalla versione 2.0.

Per verificare quale modalità è in uso::

  if layer.isUsingRendererV2():
    # nuova simbologia - sottoclasse di QgsFeatureRendererV2
    rendererV2 = layer.rendererV2()
  else:
    # vecchia simbologia - sottoclasse di QgsRenderer
    renderer = layer.renderer()

Nota: se si intende supportare le versioni più vecchie di QGIS (es. < 1.4), bisogna verificare l'esistenza del metodo :func:`isUsingRendererV2` -- in caso contrario, è possibile usare la sola vecchia simbologia::

  if not hasattr(layer, 'isUsingRendererV2'):
    print "You have an old version of QGIS"

Di seguito sarà trattata solo la nuova simbologia, che offre più opzioni di personalizzazione.

.. index:: simbologia; nuova

Nuova Simbologia
^^^^^^^^^^^^^^^^

Per avere informazioni su un visualizzatore::

  print "Type:", rendererV2.type()

La libreria QGIS Core mette a disposizione diversi tipi di visualizzatore:

=================  =======================================  ===================================================================
Tipo               Classe                                   Descrizione
=================  =======================================  ===================================================================
singleSymbol       :class:`QgsSingleSymbolRendererV2`       Visualizza tutti gli elementi con lo stesso simbolo
categorizedSymbol  :class:`QgsCategorizedSymbolRendererV2`  Visualizza gli elementi con un simbolo diverso per ogni categoria
graduatedSymbol    :class:`QgsGraduatedSymbolRendererV2`    Visualizza gli elementi con un simbolo diverso per ogni range di valori
=================  =======================================  ===================================================================

Potrebbero esserci anche visualizzatori personalizzati. Per conoscere i visualizzatori disponibili utilizzare il 
singlotone :class:`QgsRendererV2Registry`.

E' possibile ottenere un dump in formato testo dei contenuti di un visualizzatore, utile per il debugging::

  print rendererV2.dump()

.. index:: visualizzatore simbolo singolo, simbologia; visualizzatore simbolo singolo

Visualizzatore simbolo singolo
..............................

E' possibile ottenere il simbolo usato per la visualizzazione chiamando il metodo :func:`symbol`; per cambiarlo usare con il metodo :func:`setSymbol`
(nota per C++: il visualizzatore diventa proprietario del simbolo.)

.. index:: visualizzatore simbolo categorizzato, simbologia; visualizzatore simbolo categorizzato

Visualizzatore simbolo categorizzato
....................................

Per interrogare ed impostare il nome dell'attributo utilizzato per la classificazione usare i metodi: :func:`classAttribute` e :func:`setClassAttribute`.

Per ottenere la lista delle categorie::

  for cat in rendererV2.categories():
    print "%s: %s :: %s" % (cat.value().toString(), cat.label(), str(cat.symbol()))

Dove :func:`value` è il valore usato per discriminare le categorie, :func:`label` è un testo per descrivere le categorie e :func:`symbol` 
restituisce il simbolo assegnato.

Il visualizzatore solitamente memorizza anche il simbolo originale e la rampa colore usati per la classificazione:
metodi :func:`sourceColorRamp` e :func:`sourceSymbol`.

.. index:: visualizzatore simbolo graduato, simbologia; visualizzatore simbolo graduato

Visualizzatore simbolo graduato
...............................

Questo visualizzatore è molto simile al visualizzatore a simbolo categorizzato, ma
lavora per range di valori e quindi può essere usato solo con attributi numerici.

Per conoscere i range usati dal visualizzatore::

  for ran in rendererV2.ranges():
    print "%f - %f: %s %s" % (
        ran.lowerValue(), 
        ran.upperValue(), 
        ran.label(), 
        str(ran.symbol())
        )

si può usare :func:`classAttribute` per trovare il nome dell'attributo di classificazione e
i metodi :func:`sourceSymbol`  :func:`sourceColorRamp`. Il metodo :func:`mode` determina la modalità
di creazione dei range: intervalli uguali, quantili o altri metodi.

L'esempio successivo mostra come creare un proprio visualizzatore a simbolo graduato::

	from qgis.core import  (QgsVectorLayer,
                		QgsMapLayerRegistry,
				QgsGraduatedSymbolRendererV2,
		                QgsSymbolV2,
				QgsRendererRangeV2)

	myVectorLayer = QgsVectorLayer(myVectorPath, myName, 'ogr')
	myTargetField = myStyle['target_field']
	myRangeList = []
	myOpacity = 1
	# Crea il simbolo ed il range...
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
	#un altro simbolo ed un altro range
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


.. index:: simboli; lavorare con

Lavorare con i simboli
......................

Per rappresentare i simboli si usa la classe :class:`QgsSymbolV2` con tre classi derivate:

 * :class:`QgsMarkerSymbolV2` - per elementi puntuali
 * :class:`QgsLineSymbolV2` - per elementi lineari
 * :class:`QgsFillSymbolV2` - per elementi poligonali

**Ogni simbolo consiste di uno o più layer simbolo** (classi derivate da :class:`QgsSymbolLayerV2`).
I layer simbolo si occupano della visualizzazione, mentre la classe simbolo serve come contenitore dei layer simbolo.

E' possibile esplorare l'istanza di un simbolo (es. ottenuta da un visualizzatore) con il metodo :func:`type`, che ci dice se 
il simbolo è un indicatore (marker), una linea o un riempimento (fill symbol).
Il metodo :func:`dump` restituisce una breve descrizione del simbolo. Per ottenere l'elenco dei layer simbolo::

  for i in xrange(symbol.symbolLayerCount()):
    lyr = symbol.symbolLayer(i)
    print "%d: %s" % (i, lyr.layerType())

Per ottenere il colore del simbolo usare il metodo :func:`color`; per cambiare il colore del simbolo usare :func:`setColor`.

Dei simboli "indicatore" è possibile conoscere dimensione e rotazione con i metodi :func:`size` e :func:`angle`. 
Il metodo :func:`width` restituisce lo spessore di un simbolo linea.
Di default spessore e dimensione sono espressi in millimetri, gli angoli in gradi.

.. index:: layer simbolo; lavorare con

Lavorare con i layer simbolo
............................

I layer simbolo (che sono sottoclassi di :class:`QgsSymbolLayerV2`) determinano le modalità
di visualizzazione degli elementi. Vi sono diverse classi di base di layer simbolo. Allo stesso
tempo è possibile creare nuovi simboli in modo da personalizzare a piacimento la visualizzazione
degli elementi.
Il metodo :func:`layerType` identifica univocamente la classe del layer simbolo --- le classi di base sono "indicatore semplice", "linea semplice" e "riempimento semplice".

Per ottenere un elenco dei tipi di layer simbolo disponibili::

  from qgis.core import QgsSymbolLayerV2Registry
  myRegistry = QgsSymbolLayerV2Registry.instance()
  myMetadata = myRegistry.symbolLayerMetadata("SimpleFill")
  for item in myRegistry.symbolLayersForType(QgsSymbolV2.Marker): 
    print item

Output::

  EllipseMarker
  FontMarker
  SimpleMarker
  SvgMarker
  VectorField

La classe :class:`QgsSymbolLayerV2Registry` gestisce una banca dati di tutti i tipi di layer simbolo disponibili.

Per accedere ai dati di un layer simbolo, usare il metodo :func:`properties`, che restituisce il dizionario delle proprietà che determinano le modalità di visualizzazione.  
Ogni tipo di layer simbolo ha uno specifico set di proprietà. Inoltre, sono disponibili i metodi generici :func:`color`, :func:`size`, :func:`angle`, :func:`width`. Dimensione e angolo di rotazione sono disponibili solo per i simboli indicatore; spessore solo per i simboli linea.

.. index:: layer simbolo; creare tipo personalizzato

Creare un tipo personalizzato di layer simbolo
..............................................

E' possibile creare la propria classe layer simbolo per personalizzare le modalità di visualizzazione degli elementi. Segue l'esempio di un indicatore che disegna un cerchio rosso con uno specifico raggio::

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

Il metodo :func:`layerType` determina il nome del layer simbolo: deve essere univoco tra tutti i layer simbolo. Le proprietà sono usare per la persistenza degli attributi. Il metodo :func:`clone` restituisce una copia del layer simbolo con tutti gli attributi uguali.
Il metodo :func:`startRender` è chiamato prima di iniziare la visualizzazione del primo elemento, il metodo :func:`stopRender` quando la visualizzazione è completa. Il metodo :func:`renderPoint` effettua la visualizzazione. Le coordinate del punto sono sempre trasformate nelle coordinate di output.

Per le polilinee ed i poligoni cambia il metodo della visualizzazione: :func:`renderPolyline`, che riceve una lista di linee, :func:`renderPolygon` che riceve una lista di punti sul bordo esterno come primo parametro ed una lista per il bordo interno come secondo parametro.

Di solito conviene aggiungere una GUI per l'impostazione degli attributi del tipo di layer simbolo: nell'esempio precedente, potremmo permettere all'utente di impostare il raggio del cerchio. Il seguente codice implementa un widget di questo genere::

  class FooSymbolLayerWidget(QgsSymbolLayerV2Widget):
    def __init__(self, parent=None):
      QgsSymbolLayerV2Widget.__init__(self, parent)
 
      self.layer = None
 
      # imposta una semplice UI
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

Il widget può essere integrato nella finestra di dialogo delle proprietà del simbolo: quando il tipo di layer simbolo è selezionato, esso crea un'istanza del layer simbolo ed un'istanza del widget. Quindi chiama il metodo :func:`setSymbolLayer` per assegnare il layer simbolo al widget. Il widget aggiorna la UI per riflettere gli attributi del layer simbolo. La funzione :func:`symbolLayer` è usata per richiamare di nuovo il layer simbolo dal dialogo delle proprietà ed usarlo per il simbolo.

Dopo ogni cambiamento di attributo, il widget dovrebbe dare il segnale :func:`changed()` per permettere al dialogo delle proprietà di aggiornare l'anteprima del simbolo.

Ora manca il collante finale, per informare QGIS dell'esistenza di queste nuove classi: si aggiunge il layer simbolo al registro.
E' possibile usare il layer simbolo senza aggiungerlo al registro, ma alcune funzioni non saranno disponibili: es. caricare file di progetto con il layer simbolo personalizzato o modificare gli attributi del layer nella GUI.

Bisogna anche creare i metadati del layer simbolo::

  class FooSymbolLayerMetadata(QgsSymbolLayerV2AbstractMetadata):
 
    def __init__(self):
      QgsSymbolLayerV2AbstractMetadata.__init__(self, "FooMarker", QgsSymbolV2.Marker)
 
    def createSymbolLayer(self, props):
      radius = float(props[QString("radius")]) if QString("radius") in props else 4.0
      return FooSymbolLayer(radius)
 
    def createSymbolLayerWidget(self):
      return FooSymbolLayerWidget()
 
  QgsSymbolLayerV2Registry.instance().addSymbolLayerType( FooSymbolLayerMetadata() )

Bisogna passare al costruttore della classe padre il tipo di layer (lo stesso restituito dal layer) ed il tipo di simbolo (indiatore/linea/riempimento).
:func:`createSymbolLayer` si occupa di creare un'istanza del layer simbolo con gli attributi del dizionario `props`.
(Attenzione, le chiavi sono istanze di QString e non oggetti "str").
Il metodo :func:`createSymbolLayerWidget` restituisce il widget delle impostazioni per il tipo di layer simbolo.

L'ultimo passo aggiunge al registro il layer simbolo.

.. index:: 
  pair: personalizzato; visualizzatori

Creare visualizzatori personalizzati
....................................

Se si vogliono personalizzare le regole su come selezionare i simboli per la visualizzazione degli elementi (es. il simbolo è determinato come combinazione di campi, la dimensione dei simboli varia in funzione della scala di rappresentazione), bisogna implementare un nuovo visualizzatore.

Il codice seguente mostra come ottenere un visualizzatore che crea due simboli "indicatore" e sceglie in maniera random quale assegnare ai vari elementi::

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

Il costruttore della classe padre :class:`QgsFeatureRendererV2` ha bisogno di un nome per il visualizzatore (che deve essere univoco).
Il metodo :func:`symbolForFeature` definisce quale simbolo utilizzare per uno specifico elemento.
:func:`startRender` e :func:`stopRender` si occupano di inizializzare/finalizzare la visualizzazione del simbolo.
Il metodo :func:`usedAttributes` può restituire una lista di nomi di campo di cui il visualizzatore si aspetta l'esistenza.
Infine la funzione :func:`clone` restituisce una copia del visualizzatore.

Come con i layer simbolo, è possibile utilizzare una GUI per la configurazione del visualizzatore: classe :class:`QgsRendererV2Widget`. 
Il codice seguente crea un pulsante che permette all'utente di impostare il simbolo del primo simbolo::

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

Il costruttore riceve un'istanza del layer attivo (:class:`QgsVectorLayer`), lo stile globale (:class:`QgsStyleV2`) ed il visualizzatore corrente.
Se non esiste visualizzatore oppure il visualizzatore presente è di tipo diverso, sarà utilizzato il nuovo visualizzatore, altrimenti si utilizzerà il visualizzatore corrente, che è già del tipo necessario. Il contenuto del widget deve essere aggiornato per mostrare lo stato corrente del visualizzatore.
Il metodo del widget :func:`renderer` è chiamato per ottenere il visualizzatore corrente --- che sarà assegnato al layer.

Mancano i metadati del visualizzatore e la registrazione nel registro, altrimenti non sarà possibile caricare layer con il visualizzatore e l'utente non potrà selezionare il visualizzatore tra la lista di quelli disponibili.
Quindi per completare::

  class RandomRendererMetadata(QgsRendererV2AbstractMetadata):
    def __init__(self):
      QgsRendererV2AbstractMetadata.__init__(self, "RandomRenderer", "Random renderer")
 
    def createRenderer(self, element):
      return RandomRenderer()
    def createRendererWidget(self, layer, style, renderer):
      return RandomRendererWidget(layer, style, renderer)
 
  QgsRendererV2Registry.instance().addRenderer(RandomRendererMetadata())

Il costruttore dei metadati richiede un nome per il visualizzatore, un nome visibile per gli utenti ed opzionalmente un'icona.
Il metodo :func:`createRenderer` passa l'istanza :class:`QDomElement` che può essere usata per ripristinare lo stato del visualizzatore dall'albero DOM.
Il metodo :func:`createRendererWidget` crea il widget di configurazione: non è necessario che sia presente oppure può restituire `None` se il visualizzatore è sprovvisto di GUI.

Per associare un'icona al visualizzatore, essa va passata come terzo argomento al costruttore :class:`QgsRendererV2AbstractMetadata` -- 
il costruttore di classe base nella funzione RandomRendererMetadata __init__ diventa::

     QgsRendererV2AbstractMetadata.__init__(self, 
         "RandomRenderer", 
         "Random renderer",
         QIcon(QPixmap("RandomRendererIcon.png", "png")) )

L'icona può essere associata in un secondo momento usando il metodo :func:`setIcon` della classe "metadata".
L'icona può essere caricata da un file o da `Qt resource <http://qt.nokia.com/doc/4.5/resources.html>`_ (PyQt4 include un compilatore .qrc per Python).

Ulteriori Argomenti
...................

**TODO:**
 * creating/modifying symbols
 * working with style (:class:`QgsStyleV2`)
 * working with color ramps (:class:`QgsVectorColorRampV2`)
 * rule-based renderer
 * exploring symbol layer and renderer registries

.. index:: simbologia; vecchia

Vecchia simbologia
^^^^^^^^^^^^^^^^^^

Un simbolo determina colore, dimensione ed altre proprietà di un elemento.
Il visualizzatore associato al layer definisce quale simbolo usare per uno specifico elemento. 
Sono quattro i visualizzatori disponibile:

* simbolo singolo (:class:`QgsSingleSymbolRenderer`) --- tutti gli elementi sono visualizzati con lo stesso simbolo.
* valore unico (:class:`QgsUniqueValueRenderer`) --- il simbolo è scelto in funzione del valore dell'attributo.
* simbolo graduato (:class:`QgsGraduatedSymbolRenderer`) --- un simbolo per ogni gruppo (classe) di elementi, calcolato su un campo numerico
* colore continuo (:class:`QgsContinuousSymbolRenderer`)

Come creare un simbolo punto::

  sym = QgsSymbol(QGis.Point)
  sym.setColor(Qt.black)
  sym.setFillColor(Qt.green)
  sym.setFillStyle(Qt.SolidPattern)
  sym.setLineWidth(0.3)
  sym.setPointSize(3)
  sym.setNamedPointSymbol("hard:triangle")

Il metodo :func:`setNamedPointSymbol` determina la forma del simbolo. Ci sono due classi:
simboli hardcoded (prefisso ``hard:``) e simboli SVG (prefisso ``svg:``). Sono disponibili i seguenti simboli hardcoded: ``circle``, ``rectangle``, ``diamond``, ``pentagon``, ``cross``, ``cross2``, ``triangle``, ``equilateral_triangle``, ``star``, ``regular_star``, ``arrow``.

Come creare un simbolo SVG::

  sym = QgsSymbol(QGis.Point)
  sym.setNamedPointSymbol("svg:Star1.svg")
  sym.setPointSize(3)

I simboli SVG non supportano l'impostazione di colore, riempimento (fill) e stili linea.

Come creare un simbolo lineare::

  TODO

Come creare un simbolo campitura::

  TODO

Creare un visualizzatore a simbolo singolo::

  sr = QgsSingleSymbolRenderer(QGis.Point)
  sr.addSymbol(sym)

Assegnare il visualizzatore ad un layer::

  layer.setRenderer(sr)

Creare un visualizzatore a valore univoco::

  TODO

Creare un visualizzatore a simbolo graduato::

    # impostazione del campo numerico ed il numero di classi da generare
    fieldName = "My_Field"
    numberOfClasses = 5
    
    # acquisizione dell'indice del campo sulla base del nome del campo
    fieldIndex = layer.fieldNameIndex(fieldName)

    # creazione dell'oggetto visualizzatore da associare al layer
    renderer = QgsGraduatedSymbolRenderer( layer.geometryType() )

    # qui si può impostare il metodo di visualizzazione a scelta tra EqualInterval/Quantile/Empty
    renderer.setMode( QgsGraduatedSymbolRenderer.EqualInterval ) 

    # definizione delle classi (range ed etichette)
    provider = layer.dataProvider()
    minimum = provider.minimumValue( fieldIndex ).toDouble()[ 0 ]
    maximum = provider.maximumValue( fieldIndex ).toDouble()[ 0 ]

    for i in range( numberOfClasses ):
        # Switch if attribute is int or double
        lower = ('%.*f' % (2, minimum + ( maximum - minimum ) / numberOfClasses * i ) )
        upper = ('%.*f' % (2, minimum + ( maximum - minimum ) / numberOfClasses * ( i + 1 ) ) )
        label = "%s - %s" % (lower, upper)
        color = QColor(255*i/numberOfClasses, 0, 255-255*i/numberOfClasses)
        sym = QgsSymbol( layer.geometryType(), lower, upper, label, color )
        renderer.addSymbol( sym )

    # impostare l'indice campo da classificare e l'oggetto visualizzatore al layer
    renderer.setClassificationField( fieldIndex )

    layer.setRenderer( renderer )
