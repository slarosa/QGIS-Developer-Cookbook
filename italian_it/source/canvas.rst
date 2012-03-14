.. index:: area mappa

.. _canvas:

Usare l'area mappa
====================

Il widget "area mappa" è probabilmente il più importante, in quanto mostra la mappa 
ottenuta dalla sovrapposizione dei vari layer di dati e permette di interagire con mappa e layer stessi.
L'area mostra sempre una parte della mappa, così come definita dall'estensione corrente della stessa.
L'interazione è garantita dagli **strumenti di mappa**: spostamento, zoom, identificazione,
misura, modifica di vettori, etc.

L'area mappa è implementata come classe :class:`QgsMapCanvas` nel modulo :mod:`qgis.gui`. 
L'implementazione si basa sul framework Qt Graphics View, che fornisce una superficie ed una vista
che ospitano la grafica con cui l'utente può interagire. Per informazioni più dettagliate sui concetti
di scena grafica, vista ed elementi si guardi `overview of the framework <http://doc.qt.nokia.com/graphicsview.html>`_.

Ogni volta che vi è un'interazione con la mappa (zoom in/out, spostamento, etc.), la sua visualizzazione 
viene aggiornata. I layer sono rappresentati come un'immagine (usando la classe :class:`QgsMapRenderer`), che viene
visualizzata nell'area mappa. L'elemento (come inteso nel framework Qt graphics view) che si occupa di mostrare
la mappa è la classe :class:`QgsMapCanvasMap`, che allo stesso tempo controlla anche l'aggiornamento (refresh).
Oltre a questi elementi, che fanno da sfondo, ci possono essere altri **elementi area mappa**, come ad esempio
le bande elastiche (rubber bands) o gli indicatori dei vertici (vertex markers). Gli elementi dell'area sono
solitamente usati per fornire un feedback visuale degli strumenti di mappa: ad esempio, quando si crea un
nuovo poligono, lo strumento di mappa crea un elemento banda elastica che mostra la forma corrente del poligono stesso.
Tutti gli elementi area mappa sono sottoclassi di :class:`QgsMapCanvasItem`, che offre più funzionalità rispetto agli oggetti
``QGraphicsItem``.

.. index:: area mappa; architettura

L'architettura dell'area mappa consta di tre concetti:

* area mappa --- per visualizzare la mappa,
* elementi area mappa --- elementi addizionali visualizzabili nell'area mappa,
* strumenti di mappa --- per l'interazione con l'area mappa.

.. index:: area mappa; integrare

Widget area mappa
-----------------

L'area mappa è un widget come ogni altro widget delle Qt, per cui si crea ed usa facilmente con::

  canvas = QgsMapCanvas()
  canvas.show()

Il codice produce una finestra standalone con un'area mappa, che può anche essere integrat
in un widget o finestra esistente. Se si utilizzano files .ui ed il Qt designer,
posizionare ``QWidget`` sulla form e promuoverlo a nuova classe: impostare ``QgsMapCanvas`` 
come nome della classe e ``qgis.gui`` come header file. L'utility ``pyuic4`` si occuperà del tutto. 
Questa è la maniera più conveniente per integrare l'area mappa. Un'altra modalità consiste nello
scrivere manualmente il codice per costruire l'area mappa e gli altri widget (come figli di una finestra
principale o di un dialogo) e creare un layout.

Di default, l'area mappa ha uno sfondo nero e non usa l'anti-aliasing. Per impostare lo sfondo bianco 
ed abilitare l'anti-aliasing::

  canvas.setCanvasColor(Qt.white)
  canvas.enableAntiAliasing(True)

(``Qt`` viene dal modulo ``PyQt4.QtCore`` e ``Qt.white`` è una delle istanze ``QColor`` predefinite.)

Ora è tempo di aggiungere qualche layer. Come prima cosa apriremo un layer e lo
aggiungeremo al registro dei layer mappa, di seguito imposteremo l'estensione dell'area mappa 
e l'elenco dei layer::

  layer = QgsVectorLayer(path, name, provider)
  if not layer.isValid():
    raise IOError, "Failed to open the layer"

  # aggiunge il layer al registro
  QgsMapLayerRegistry.instance().addMapLayer(layer)

  # imposta l'estensione dell'area all'estensione del layer
  canvas.setExtent(layer.extent())

  # imposta i layer per l'area mappa
  canvas.setLayerSet( [ QgsMapCanvasLayer(layer) ] )

Ad esecuzione di questi comandi, l'area dovrebbe mostrare i layer caricati.

.. index:: area mappa; strumenti mappa

Utilizzare gli strumenti mappa
------------------------------

L'esempio seguente costruisce una finestra contenente un'area mappa e strumenti 
di base per il pan e lo zoom: lo strumento spostamento è creato con la classe :class:`QgsMapToolPan`, 
lo strumento zoom con un paio di istanze della classe:class:`QgsMapToolZoom`. Le azioni sono impostate 
a "checkable" e di seguito assegnate agli strumenti per permettere la gestione automatica dello stato 
"checked/unchecked" delle azioni stesse -- quando uno strumento viene attivato, la sua azione è marcata 
come selezionata e l'azione dello strumento precedente viene deselezionata. 
Gli strumenti mappa sono attivati con il metodo :func:`setMapTool`.

::


  from qgis.gui import *
  from PyQt4.QtGui import QAction, QMainWindow
  from PyQt4.QtCore import SIGNAL, Qt, QString

  class MyWnd(QMainWindow):
    def __init__(self, layer):
      QMainWindow.__init__(self)

      self.canvas = QgsMapCanvas()
      self.canvas.setCanvasColor(Qt.white)

      self.canvas.setExtent(layer.extent())
      self.canvas.setLayerSet( [ QgsMapCanvasLayer(layer) ] )

      self.setCentralWidget(self.canvas)
      
      actionZoomIn = QAction(QString("Zoom in"), self)
      actionZoomOut = QAction(QString("Zoom out"), self)
      actionPan = QAction(QString("Pan"), self)
      
      actionZoomIn.setCheckable(True)
      actionZoomOut.setCheckable(True)
      actionPan.setCheckable(True)
      
      self.connect(actionZoomIn, SIGNAL("triggered()"), self.zoomIn)
      self.connect(actionZoomOut, SIGNAL("triggered()"), self.zoomOut)
      self.connect(actionPan, SIGNAL("triggered()"), self.pan)

      self.toolbar = self.addToolBar("Canvas actions")
      self.toolbar.addAction(actionZoomIn)
      self.toolbar.addAction(actionZoomOut)
      self.toolbar.addAction(actionPan)

      # crea gli strumenti mappa
      self.toolPan = QgsMapToolPan(self.canvas)
      self.toolPan.setAction(actionPan)
      self.toolZoomIn = QgsMapToolZoom(self.canvas, False) # false = in
      self.toolZoomIn.setAction(actionZoomIn)
      self.toolZoomOut = QgsMapToolZoom(self.canvas, True) # true = out
      self.toolZoomOut.setAction(actionZoomOut)
      
      self.pan()

    def zoomIn(self):
      self.canvas.setMapTool(self.toolZoomIn)

    def zoomOut(self):
      self.canvas.setMapTool(self.toolZoomOut)

    def pan(self):
      self.canvas.setMapTool(self.toolPan)

Si può inserire il codice precedente in un file, es. ``mywnd.py``, e provarlo 
nella console python di QGIS.
Il codice seguente mette il layer correntemente selezionato in una nuova area mappa::

  import mywnd
  w = mywnd.MyWnd(qgis.utils.iface.activeLayer())
  w.show()

Assicurarsi che il file ``mywnd.py`` sia posizionato all'interno del percorso di 
ricerca di python (``sys.path``). Per aggiungerlo al percorso, nel caso non lo sia:
``sys.path.insert(0,'/my/path')``.

.. index:: area mappa; bande elastiche, area mappa; indicatori vertici

Bande elastiche ed indicatori dei vertici
-----------------------------------------

Per mostrare altre informazioni nell'area mappa si usano gli "elementi area mappa".
E' possibile creare degli elementi personalizzati; quelli disponibili sono:
:class:`QgsRubberBand` per disegnare poli-linee e poligoni e :class:`QgsVertexMarker` 
per disegnare punti. Lavorano entrambi in coordinate di mappa, in modo tale che la forma
è automaticamente spostata/scalata quando l'area mappa viene spostata e/o ingrandita/rimpicciolita.

Per mostrare una poli-linea::

  r = QgsRubberBand(canvas, False)  # False = not a polygon
  points = [ QgsPoint(-1,-1), QgsPoint(0,1), QgsPoint(1,-1) ]
  r.setToGeometry(QgsGeometry.fromPolyline(points), None)

Per mostrare un poligono::

  r = QgsRubberBand(canvas, True)  # True = a polygon
  points = [ [ QgsPoint(-1,-1), QgsPoint(0,1), QgsPoint(1,-1) ] ]
  r.setToGeometry(QgsGeometry.fromPolygon(points), None)

Si noti che i punti per i poligoni non sono un semplice elenco: infatti, 
essi sono una lista  contenente gli anelli del poligono: il primo anello costituisce
il bordo esterno, altri anelli (opzionali) corrispondono a buchi nel poligono.

Delle bande elastiche è possibile modificare colore e spessore della linea::

  r.setColor(QColor(0,0,255))
  r.setWidth(3)

Gli elementi sono vincolati all'area mappa. Per nasconderli temporaneamente (e/o mostrarli di nuovo) 
usare :func:`hide` e :func:`show`; per rimuoverli completamente usare::

  canvas.scene().removeItem(r)

(in C++ è possibile eliminare gli elementi, mentre in Python ``del r`` eliminerebbe giusto il riferimento
e l'oggetto continuerebbe ad esistere in quanto appartiene alla vista)

Una banda elastica può servire a disegnare punti, anche se è preferibile usare la classe 
:class:`QgsVertexMarker` (:class:`QgsRubberBand` disegnerebbe giusto un rettangolo intorno al punto di interesse).
Per usare un indicatore di vertice::

  m = QgsVertexMarker(canvas)
  m.setCenter(QgsPoint(0,0))

Il codice disegna una croce rossa sulla posizione [0,0]; è possibile personalizzare tipo,
dimensione, colore e spessore della penna::

  m.setColor(QColor(0,255,0))
  m.setIconSize(5)
  m.setIconType(QgsVertexMarker.ICON_BOX) # or ICON_CROSS, ICON_X
  m.setPenWidth(3)

Per nascondere temporaneamente o eliminare gli indicatori di vertice si procede come per le bande elastiche.

.. index:: area mappa; implementare uno strumento di mappa persomnalizzato

Scrivere strumenti di Mappa personalizzati
------------------------------------------

**TODO:** how to create a map tool

.. index:: area mappa; implementare oggetti per l'area di mappa

Scrivere oggetti per l'area di mappa personalizzati
---------------------------------------------------

**TODO:** how to create a map canvas item



.. TODO - custom application example?
  from qgis.core import QgsApplication
  from qgis.gui import QgsMapCanvas
  import sys
  def init():
    a = QgsApplication(sys.argv, True)
    QgsApplication.setPrefixPath('/home/martin/qgis/inst', True)
    QgsApplication.initQgis()
    return a
  def show_canvas(app):
    canvas = QgsMapCanvas()
    canvas.show()
    app.exec_()
  app = init()
  show_canvas(app)
