
.. _network-analysis:

Libreria per l'analisi delle reti
=================================

A partire dalla versione 1.8, QGIS ha integrato tra le sue librerie di analisi *core* la libreria per l'analisi delle reti (network analysis).
La libreria:

 * crea un grafo matematico a partire da dati geografici (layer vettoriali di polilinee)
 * implementa dei metodi di base della teoria dei grafi (algoritmo di Dijkstra, per il momento)

La libreria è stata creata esportando le funzionalità di base del plugin RoadGraph: ora è possibile
usarne i metodi per lo sviluppo di plugin o direttamente nella console Python.

Informazioni generali
---------------------

Un tipico caso d'uso potrebbe essere il seguente:

1. creare un grafo a partire da dati geografici vettoriali (polilinee)
2. lanciare l'analisi del grafo
3. usare i risultati dell'analisi (visualizzandoli, ad esempio)

Costruire un grafo
------------------

La prima cosa da fare è preparare i dati di input, cioè convertire dei dati vettoriali in un grafo.
Tutte le operazioni successive useranno questo grafo e non il layer di partenza.

Come dati di input è possibile usare un layer di polilinee. I nodi della polilinea diventano i vertici del grafo,
i segmenti di polilinea gli spigoli (archi, lati) del grafo stesso.
Nodi con stesse coordinate risultano nello stesso vertice di grafo, per cui
due linee che condividono un nodo risultano connesse.

Inoltre, durante la creazione del grafo è possibile agganciare punti addizionali al layer vettoriale di input.
Per ogni punto addizionale verrà individuata una corrispondenza --- vertice di grafo più vicino o spigolo più vicino.
In caso di spigolo, lo stesso sarà diviso in due in seguito all'aggiunta del nuovo vertice.
   
Come proprietà di uno spigolo possono essere utilizzate la lunghezza dello stesso e gli attributi del layer vettoriale.

Il convertitore da layer vettoriale a grafo è sviluppato con il modello di programmazione cosiddetto `Builder <http://it.wikipedia.org/wiki/Builder_pattern>`_ tramite la classe Director. Allo stato attuale è disponibile il solo Director: `QgsLineVectorLayerDirector <http://doc.qgis.org/api/classQgsLineVectorLayerDirector.html>`_.
Il *director* gestisce le impostazioni di base per la costruzione del grafo a partire da layer vettoriali di linee: tali impostazioni saranno usate dal 
*builder* per costruire il grafo. Esiste attualmente un solo *builder*: `QgsGraphBuilder <http://doc.qgis.org/api/classQgsGraphBuilder.html>`_,
che crea oggetti `QgsGraph <http://doc.qgis.org/api/classQgsGraph.html>`_.
Si potrebbe implementare un proprio *builder* compatibile con librerie tipo `BGL <http://www.boost.org/doc/libs/1_48_0/libs/graph/doc/index.html>`_
e `NetworkX <http://networkx.lanl.gov/>`_.

Per calcolare le proprietà degli spigoli viene utilizzato il modello di programmazione `strategy <http://it.wikipedia.org/wiki/Strategy_pattern>`_.
Per ora è disponibile la sola *strategy* `QgsDistanceArcProperter <http://doc.qgis.org/api/classQgsDistanceArcProperter.html>`_: tiente conto della lunghezza. 
E' possibile implementare una propria *strategy*. Ad esempio, il plugin RoadGraph usa una *strategy* che calcola il tempo di percorrenza utilizzando la lunghezze degli spigoli ed una valore di velocità dagli attributi.

Per usare la libreria bisogna importare il modulo *networkanalysis*::

    from qgis.networkanalysis import *

Quindi creare il *director*::

       # non utilizza le informazioni sul senso di marcia dagli attributi
       # tutte le strade sono considerate a doppio senso
       director = QgsLineVectorLayerDirector( vLayer, -1, '', '', '', 3 )

       # usa il campo con indice 5 come fonte di informazioni sul senso di marcia
       # strade a senso unico con senso di marcia uguale alla sequenza dei vertici hanno un attributo con valore "yes",
       # quelle con senso di marcia inverso alla sequenza dei vertici hanno valore pari a — "1". Le strade a doppio senso
       # hanno un attrobuto pari a  — "no". Di default, le strade sono a doppio senso. Questo schema è compatibile con 
       # il modello dati di OpenStreetMap
       director = QgsLineVectorLayerDirector( vLayer, 5, 'yes', '1', 'no', 3 )

Per costruire un *director* dobbiamo passare il layer vettoriale da usare come fonte
per il grafo e per le informazioni sui vari movimenti possibili su ogni segmento di strada
(senso unico o doppio senso, direzione diretta o inversa).
Segue la lista dei parametri:

 * **vl** — layer vettoriale da usare per la costruzione del grafo
 * **directionFieldId** — indice del campo attributo in cui sono presenti le informazioni sul senso di marcia. Se impostato a -1, le informazioni non vengono utilizzate
 * **directDirectionValue** — valore del campo per le strade con direzione diretta (movimento dal primo punto della linea all'ultimo)
 * **reverseDirectionValue** — valore del campo per le strade con direzione inversa (movimento dall'ultimo punto della linea verso il primo)
 * **bothDirectionValue** — valore del campo per le strade a doppio senso
 * **defaultDirection** — valore di default per la direzione delle strade. Il valore sarà usato per quelle strade prive di valore **directionFieldId**

In seguito, bisogna creare una *strategy* per calcolare le proprietà degli spigoli del grafo::

  properter = QgsDistanceArcProperter()

Quindi, passare tale *strategy* al *director*::

  director.addProperter( properter )

Ora è possibile creare il *builder* per la creazione del grafo. 
Il costruttore QgsGraphBuilder prende diversi argomenti:

* **crs** — sistema di riferimento delle coordinate. Argomento obligatorio
* **otfEnabled** — usare o meno la riproiezione «on the fly». Default = true (usa la OTF)
* **topologyTolerance** — tolleranza topologica. Default = 0
* **ellipsoidID** — ellissoide da utilizzare. Default = WGS84.

::

  # solo CRS impostato: gli altri argomentio prendono i valori di default
  builder = QgsGraphBuilder( myCRS )

E' possibile impostare diversi punti da utilizzare nell'analisi. Ad esempio::

  startPoint = QgsPoint( 82.7112, 55.1672 )
  endPoint = QgsPoint( 83.1879, 54.7079 )

Ora è possible costruire il grafo ed agganciare punti ad esso::

  tiedPoints = director.makeGraph( builder, [ startPoint, endPoint ] )

La costruzione del grafo può richiedere molto tempo (dipende dal numero degli elementi del layer vettoriale sorgente e dalla dimensione dello stesso).
*tiedPoints* è una lista di coordinate dei punti agganciati. Alla fine dell'operazione di costruzione, il grafo sarà disponibile per le successive analisi::

  graph = builder.graph()

Con il codice seguente è possibile acquisire gli indici dei punti::

  startId = graph.findVertex( tiedPoints[ 0 ] )
  endId = graph.findVertex( tiedPoints[ 1 ] )


Analisi del grafo
-----------------

L'analisi delle reti serve per rispondere a domande tipo: quali vertici sono connessi e come
trovare il percorso più breve? Per risolvere questo tipo di problemi la libreria 
mette a disposizione l'algoritmo di Dijkstra.

L'algoritmo di Dijkstra trova il percorso migliore da un vertice del grafo a tutti gli altri, insieme
ai parametri di ottimizzazione. Il risultato può essere rappresentato come un albero dei cammini minimi.

L'albero dei cammini minimi è un grafo pesato orientato (più precisamente un albero) con le seguenti proprietà:

  * un solo vertice non ha spigoli entranti — la radice dell'albero
  * tutti gli altri vertici hanno un solo spigolo entrante
  * se il vertice B è raggiungibile dal vertice A, allora il percorso AB è l'unico percorso disponibile ed è quello ottimale (più breve) su questo grafo

Per ottenere l'albero dei cammini minimi si usano i metodi :func:`shortestTree` e
:func:`dijkstra` della classe `QgsGraphAnalyzer <http://doc.qgis.org/api/classQgsGraphAnalyzer.html>`_.
Si raccomanda l'uso di :func:`dijkstra`, più veloce e con un uso più efficiente della memoria.

Il metodo :func:`shortestTree` è utile per navigare l'albero dei cammini minimi. Il metodo
crea un oggetto grafo (QgsGraph) ed accetta tre variabili:

  * **source** — il grafo in input
  * **startVertexIdx** — indice del punto radice dell'albero
  * **criterionNum** — numero di proprietà spigolo da usare, inziando da 0

::

  tree = QgsGraphAnalyzer.shortestTree( graph, startId, 0 )

Il metodo :func:`dijkstra` ha gli stessi argomenti, ma restituisce due *array*.
Nel primo *array* l'elemento *i* contiene l'indice dello spigolo entrante oppure *-1* in assenza si spigoli entranti.
Nel secondo *array* l'elemento *i* contiene la distanza dalla radice dell'albero al vertice *i* oppure DOUBLE_MAX 
se il vertice *i* non è raggiungibile dalla radice.

::

  (tree, cost) = QgsGraphAnalyzer.dijkstra( graph, startId, 0 )

Il codice che segue mostra la creazione di un albero dei cammini minimi
utilizzando il metodo :func:`shortestTree`. **Attenzione**: utilizzare questo codice solo come esempio;
esso crea molti oggetti `QgsRubberBand <http://doc.qgis.org/api/classQgsRubberBand.html>`_ e può essere
lento su dataset pesanti.

::

  from PyQt4.QtCore import *
  from PyQt4.QtGui import *

  from qgis.core import *
  from qgis.gui import *
  from qgis.networkanalysis import *

  vl = qgis.utils.iface.mapCanvas().currentLayer()
  director = QgsLineVectorLayerDirector( vl, -1, '', '', '', 3 )
  properter = QgsDistanceArcProperter()
  director.addProperter( properter )
  crs = qgis.utils.iface.mapCanvas().mapRenderer().destinationCrs()
  builder = QgsGraphBuilder( crs )

  pStart = QgsPoint( -0.743804, 0.22954 )
  tiedPoint = director.makeGraph( builder, [ pStart ] )
  pStart = tiedPoint[ 0 ]

  graph = builder.graph()

  idStart = graph.findVertex( pStart )

  tree = QgsGraphAnalyzer.shortestTree( graph, idStart, 0 )

  i = 0;
  while ( i < tree.arcCount() ):
    rb = QgsRubberBand( qgis.utils.iface.mapCanvas() )
    rb.setColor ( Qt.red )
    rb.addPoint ( tree.vertex( tree.arc( i ).inVertex() ).point() )
    rb.addPoint ( tree.vertex( tree.arc( i ).outVertex() ).point() )
    i = i + 1

Stessa cosa ma usando il metodo :func:`dijkstra`::

  from PyQt4.QtCore import *
  from PyQt4.QtGui import *

  from qgis.core import *
  from qgis.gui import *
  from qgis.networkanalysis import *

  vl = qgis.utils.iface.mapCanvas().currentLayer()
  director = QgsLineVectorLayerDirector( vl, -1, '', '', '', 3 )
  properter = QgsDistanceArcProperter()
  director.addProperter( properter )
  crs = qgis.utils.iface.mapCanvas().mapRenderer().destinationCrs()
  builder = QgsGraphBuilder( crs )

  pStart = QgsPoint( -1.37144, 0.543836 )
  tiedPoint = director.makeGraph( builder, [ pStart ] )
  pStart = tiedPoint[ 0 ]

  graph = builder.graph()

  idStart = graph.findVertex( pStart )

  ( tree, costs ) = QgsGraphAnalyzer.dijkstra( graph, idStart, 0 )

  for edgeId in tree:
    if edgeId == -1:
      continue
    rb = QgsRubberBand( qgis.utils.iface.mapCanvas() )
    rb.setColor ( Qt.red )
    rb.addPoint ( graph.vertex( graph.arc( edgeId ).inVertex() ).point() )
    rb.addPoint ( graph.vertex( graph.arc( edgeId ).outVertex() ).point() )

Trovare il percorso minimo
^^^^^^^^^^^^^^^^^^^^^^^^^^

Il calcolo del percorso ottimale tra due punti si basa sul seguente approccio.
Entrambi i punti (inizio A e fine B) sono agganciati al grafo durante la sua costruzione.
Con i metodi :func:`shortestTree` oppure :func:`dijkstra` si costruisce l'albero dei cammini minimi con A come radice.
Nello stesso albero si individua il punto B e si inizia a navigare l'albero da B verso A.
Segue un esempio di pseudocodice dell'algoritmo::

    assign Т = B
    while Т != A
        add point Т to path
        get incoming edge for point Т
        look for point ТТ, that is start point of this edge
        assign Т = ТТ
    add point А to path

A questo punto si ha a disposizione il percorso, sotto forma di lista invertita dei vertici 
(i vertici sono elencati in ordine inverso, dal punto di fine al punto di partenza) visitati dal percorso stesso. 

Segue codice di esempio per la console Python di QGIS (bisogna selezionare un layer di linee in legenda ed utilizzare due punti opportuni);
viene usato il metodo :func:`shortestTree`::

  from PyQt4.QtCore import *
  from PyQt4.QtGui import *

  from qgis.core import *
  from qgis.gui import *
  from qgis.networkanalysis import *

  vl = qgis.utils.iface.mapCanvas().currentLayer()
  director = QgsLineVectorLayerDirector( vl, -1, '', '', '', 3 )
  properter = QgsDistanceArcProperter()
  director.addProperter( properter )
  crs = qgis.utils.iface.mapCanvas().mapRenderer().destinationCrs()
  builder = QgsGraphBuilder( crs )

  pStart = QgsPoint( -0.835953, 0.15679 )
  pStop = QgsPoint( -1.1027, 0.699986 )

  tiedPoints = director.makeGraph( builder, [ pStart, pStop ] )
  graph = builder.graph()

  tStart = tiedPoints[ 0 ]
  tStop = tiedPoints[ 1 ]

  idStart = graph.findVertex( tStart )
  tree = QgsGraphAnalyzer.shortestTree( graph, idStart, 0 )

  idStart = tree.findVertex( tStart )
  idStop = tree.findVertex( tStop )

  if idStop == -1:
    print "Path not found"
  else:
    p = []
    while ( idStart != idStop ):
      l = tree.vertex( idStop ).inArc()
      if len( l ) == 0:
        break
      e = tree.arc( l[ 0 ] )
      p.insert( 0, tree.vertex( e.inVertex() ).point() )
      idStop = e.outVertex()

    p.insert( 0, tStart )
    rb = QgsRubberBand( qgis.utils.iface.mapCanvas() )
    rb.setColor( Qt.red )

    for pnt in p:
      rb.addPoint(pnt)

La stessa cosa, ma usando il metodo :func:`dikstra`::

  from PyQt4.QtCore import *
  from PyQt4.QtGui import *

  from qgis.core import *
  from qgis.gui import *
  from qgis.networkanalysis import *

  vl = qgis.utils.iface.mapCanvas().currentLayer()
  director = QgsLineVectorLayerDirector( vl, -1, '', '', '', 3 )
  properter = QgsDistanceArcProperter()
  director.addProperter( properter )
  crs = qgis.utils.iface.mapCanvas().mapRenderer().destinationCrs()
  builder = QgsGraphBuilder( crs )

  pStart = QgsPoint( -0.835953, 0.15679 )
  pStop = QgsPoint( -1.1027, 0.699986 )

  tiedPoints = director.makeGraph( builder, [ pStart, pStop ] )
  graph = builder.graph()

  tStart = tiedPoints[ 0 ]
  tStop = tiedPoints[ 1 ]

  idStart = graph.findVertex( tStart )
  idStop = graph.findVertex( tStop )

  ( tree, cost ) = QgsGraphAnalyzer.dijkstra( graph, idStart, 0 )

  if tree[ idStop ] == -1:
    print "Path not found"
  else:
    p = []
    curPos = idStop
    while curPos != idStart:
      p.append( graph.vertex( graph.arc( tree[ curPos ] ).inVertex() ).point() )
      curPos = graph.arc( tree[ curPos ] ).outVertex();

    p.append( tStart )

    rb = QgsRubberBand( qgis.utils.iface.mapCanvas() )
    rb.setColor( Qt.red )

    for pnt in p:
      rb.addPoint(pnt)

Aree di disponibilità
^^^^^^^^^^^^^^^^^^^^^

L'area di disponibilità di un vertice A è un sottoinsieme di vertici del grafo accessibili da A: il costo del percorso verso
questi vertici è non maggiore di un valore dato.

Facciamo un esempio per meglio comprendere il concetto: "Questa è una caserma di vigili del fuoco.
Quale parte di città può essere raggiunta in 5 minuti? Ed in 10? Ed in 15?"
Le risposte a queste domande costituiscono le aree di disponibilità della caserma.
 
Per trovare le aree di disponibilità si usa il metodo :func:`dijksta` della classe :class:`QgsGraphAnalyzer`, 
sufficiente per confrontare elementi di un *array* di costo con valore predefinito. Se il costo *i* è minore o uguale al valore
predefinito, allora il vertice *i* è all'interno dell'area di disponibilità, altrimenti è all'esterno.

Più complicato è trovare i bordi di un'area di disponibilità. Il bordo inferiore è dato dai vertici ancora accessibili;
il bordo superiore da quelli non accessibili. In effetti il bordo di accessibilità passa per quei vertici dell'albero del cammino minimo
per cui il vertice di inizio è accessibile ed il vertice di fine è non accessibile.

Segue un esempio::

  from PyQt4.QtCore import *
  from PyQt4.QtGui import *

  from qgis.core import *
  from qgis.gui import *
  from qgis.networkanalysis import *

  vl = qgis.utils.iface.mapCanvas().currentLayer()
  director = QgsLineVectorLayerDirector( vl, -1, '', '', '', 3 )
  properter = QgsDistanceArcProperter()
  director.addProperter( properter )
  crs = qgis.utils.iface.mapCanvas().mapRenderer().destinationCrs()
  builder = QgsGraphBuilder( crs )

  pStart = QgsPoint( 65.5462, 57.1509 )
  delta = qgis.utils.iface.mapCanvas().getCoordinateTransform().mapUnitsPerPixel() * 1

  rb = QgsRubberBand( qgis.utils.iface.mapCanvas(), True )
  rb.setColor( Qt.green )
  rb.addPoint( QgsPoint( pStart.x() - delta, pStart.y() - delta ) )
  rb.addPoint( QgsPoint( pStart.x() + delta, pStart.y() - delta ) )
  rb.addPoint( QgsPoint( pStart.x() + delta, pStart.y() + delta ) )
  rb.addPoint( QgsPoint( pStart.x() - delta, pStart.y() + delta ) )

  tiedPoints = director.makeGraph( builder, [ pStart ] )
  graph = builder.graph()
  tStart = tiedPoints[ 0 ]

  idStart = graph.findVertex( tStart )

  ( tree, cost ) = QgsGraphAnalyzer.dijkstra( graph, idStart, 0 )

  upperBound = []
  r = 2000.0
  i = 0
  while i < len(cost):
    if cost[ i ] > r and tree[ i ] != -1:
      outVertexId = graph.arc( tree [ i ] ).outVertex()
      if cost[ outVertexId ] < r:
        upperBound.append( i )
    i = i + 1

  for i in upperBound:
    centerPoint = graph.vertex( i ).point()
    rb = QgsRubberBand( qgis.utils.iface.mapCanvas(), True )
    rb.setColor( Qt.red )
    rb.addPoint( QgsPoint( centerPoint.x() - delta, centerPoint.y() - delta ) )
    rb.addPoint( QgsPoint( centerPoint.x() + delta, centerPoint.y() - delta ) )
    rb.addPoint( QgsPoint( centerPoint.x() + delta, centerPoint.y() + delta ) )
    rb.addPoint( QgsPoint( centerPoint.x() - delta, centerPoint.y() + delta ) )
