.. index:: plugin; sviluppare, Python; sviluppare plugin

.. _plugins:

Sviluppare plugin con Python
============================

Lo sviluppo di plugin con Python risulta più agevole rispetto all'utilizzo di C++:
i plugin python sono più facili da creare, gestire e distribuire.

Tutti i plugin di QGIS, Python e C++, sono elencati nel gestore dei plugin:
di default sono memorizzati nei seguenti percorsi:

    * UNIX/Mac: :file:`~/.qgis/python/plugins` e :file:`(qgis_prefix)/share/qgis/python/plugins`
    * Windows: :file:`~/.qgis/python/plugins` e :file:`(qgis_prefix)/python/plugins`

La cartella "home" (indicata con :file:`~`) su Windows è di solito qualcosa simile a :file:`C:\\Documents and Settings\\(user)`. 
Sottocartelle di questi percorsi sono considerati come pacchetti Python e possono essere importati in un plugin di QGIS.

Fasi:

1. *Idea*: Avere un'idea circa quello che si intende fare con il plugin.
   Perchè lo si fa?
   Quale problema si vuole risolvere?
   Esiste un plugin che già tratta lo stesso problema?

2. *Creare i files*: Creare i file descritti di seguito.
   :file:`__init.py__`: punto di partenza
   :file:`plugin.py`: cortpo principale del plugin 
   :file:`form.ui` e :file:`resources.qrc`: una form in QT-Designer

3. *Scrivere il codice*: scrivere il codice nel file :file:`plugin.py`

4. *Testare*: chiudere e riaprire QGIS ed impostare il nuovo plugin. Controllare che tutto funzioni.

5. *Pubblicare*: pubblicare il plugin in un repository QGIS o distribuirlo tramite un proprio repository.

.. index:: plugin; scrivere

Scrivere un plugin
------------------

I plugin di QGIS scritti in Python sono elencati su `Plugin Repositories wiki page <http://www.qgis.org/wiki/Python_Plugin_Repositories>`_:
è possibile scaricare un plugin e studiarne il codice per imparare a programmare con PyQGIS.
Se si è pronti per sviluppare un plugin ma non si ha un'idea, controllare la lista delle funzionalità richieste dalla comunità degli utenti
su `Python Plugin Ideas wiki page <http://www.qgis.org/wiki/Python_Plugin_Ideas>`_.

.. index:: plugin; file necessari

Creare i file necessari
-----------------------

Questa che segue è la struttura della cartella per il nostro plugin di esempio::

  PYTHON_PLUGINS_PATH/
    testplug/
      __init__.py
      plugin.py
      metadata.txt
      resources.qrc
      resources.py
      form.ui
      form.py

* __init__.py = E' il punto di partenza per lo sviluppo di ogni plugin. Contiene informazioni generali, versione, nome e classi principali
* plugin.py = Il codice principale del plugin. Contiene tutte le informazioni sulle funzionalità del plugin ed il codice principale
* resources.qrc = E' il documento .xml creato con il QT-Designer. Contiene i percorsi relativi alle risorse dei form
* resources.py = La traduzione in Python del file resources.qrc
* form.ui = La GUI creata con QT-Designer
* form.py = La traduzione in Python della form.ui
* metadata.txt = Richiesto a partire da QGIS 1.8.0. Contiene informazioni generali, versione, nome ed altre informazioni usate dal sito web e dall'infrastruttura del plugin. E' preferibile inserire i metadati in "metadata.txt", piuttosto che in "__init__.py" (A partire da QGIS 2.0 l'uso di "metadata.txt" diventerà obbligatorio).

Ai due seguenti link è possibile creare in maniera automatica lo scheletro di un plugin Python: `qui <http://pyqgis.org/builder/plugin_builder.py>`_ e `qui <http://www.dimitrisk.gr/qgis/creator/>`_.
In QGIS è disponibile il plugin `Plugin Builder` che fa la stessa cosa, senza necessità di una connessione ad Internet.

.. index:: plugin; scrivere il codice

Scrivere il codice
------------------

.. index:: plugin; __init__.py, __init__.py

__init__.py
^^^^^^^^^^^

Il file :file:`__init__.py` contiene informazioni come il nome e la descrizione del plugin::

  def name():
    return "Mio plugin"
  
  def description():
    return "Questo plugin fa questo e quello."
  
  def version():
    return "Versione 0.1"
  
  def qgisMinimumVersion(): 
    return "1.0"
  
  def authorName():
    return "Sviluppatore Mario"
  
  def classFactory(iface):
    # carica la classe TestPlugin dal file testplugin.py
    from testplugin import TestPlugin
    return TestPlugin(iface)

In QGIS 1.9.90 i plugin possono essere posizionati non solo nel menu `Plugins`, ma anche nei menu `Raster`, `Vector`, `Database` e `Web`, per cui
si è introdotta la nuova voce di metadati "category".I valori che la voce può prendere sono:
Vector, Raster, Database, Web e Layers. Ad esempio, per inserire il plugin nel menu `Raster` aggiungere ad :file:`__init__.py`::

  def category():
    return "Raster"

.. index:: plugin; metadata.txt, metadati, metadata.txt

metadata.txt
^^^^^^^^^^^^

A partire da QGIS 1.8 il file metadata.txt è obbligatorio (`si veda <https://github.com/qgis/qgis-django/blob/master/qgis-app/plugins/docs/introduction.rst>`_)
Segue un esempio di metadata.txt::

  ; la sezione seguente è obbligatoria
  [general]
  name=CiaoMondo
  qgisMinimumVersion=1.8
  description=Questo è un plugin per salutare il mondo
  category=Raster
  version=version 1.2
  ; fone dei metadati obbligatori
  
  ; inizio metadati opzionali
  changelog=questo è un changelog
      molto
      molto
      molto
      molto lungo disposto su più riche
  
  ; tag separati da virgole. Spazi ammessi
  tags=wkt,raster,cia mondo
  
  ; questi metadati possono essere vuoti
  ; in una futura versione dell'applicazione web
  ; sarà probabilmente possibile creare un progetto su redmine
  ; se questimetadati non sono compilati
  homepage=http://www.itopen.it
  tracker=http://bugs.itopen.it
  repository=http://www.itopen.it/repo
  icon=icon.png
  
  ; flag sperimentale
  experimental=True
  
  ; flag deprecata (si applica all'intero whole e non solo alla versione in upload)
  deprecated=False

.. index:: plugin; plugin.py, plugin.py

plugin.py
^^^^^^^^^

La funzione ``classFactory()`` è chiamata non appena il plugin viene caricato in QGIS. Riceve riferimenti ad istanze di :`QgisInterface` 
e deve restituire istanze del plugin sviluppato - nell'esempio chiamato ``TestPlugin``. Il codice che segue mostra come dovrebbe essere
la classe (e. :file:`testplugin.py`)::

  from PyQt4.QtCore import *
  from PyQt4.QtGui import *
  from qgis.core import *

  # inizializza le risorse Qt dal file resouces.py
  import resources

  class TestPlugin:

    def __init__(self, iface):
      # salva riferimento all'interfaccia di QGIS
      self.iface = iface

    def initGui(self):
      # crea l'azione che inizializza la configurazione del plugin
      self.action = QAction(QIcon(":/plugins/testplug/icon.png"), "Test plugin", self.iface.mainWindow())
      self.action.setWhatsThis("Configuration for test plugin")
      self.action.setStatusTip("This is status tip")
      QObject.connect(self.action, SIGNAL("triggered()"), self.run)

      # agfgiunge pulsante nella barra degli strumenti e la vove di menu
      self.iface.addToolBarIcon(self.action)
      self.iface.addPluginToMenu("&Test plugins", self.action)

      # connessione al segnale renderComplete emesso in seguito alla visualizzazione dell'area mappa
      QObject.connect(self.iface.mapCanvas(), SIGNAL("renderComplete(QPainter *)"), self.renderTest)

    def unload(self):
      # rimuove la voce di menu e l'icona del plugin
      self.iface.removePluginMenu("&Test plugins",self.action)
      self.iface.removeToolBarIcon(self.action)

      # disconnessione dal segnale dell'area mappa
      QObject.disconnect(self.iface.MapCanvas(), SIGNAL("renderComplete(QPainter *)"), self.renderTest)

    def run(self):
      # crea e mostra un dialogo di configurazione
      print "TestPlugin: run called!"

    def renderTest(self, painter):
      # usa painter per disegnare nell'area mappa
      print "TestPlugin: renderTest called!"

A partire da QGIS 1.9.90, per inserire il plugin come voce dei nuovi menu `Raster`, `Vector`, `Database` o `Web`, bisogna
modificare il codice delle funzioni ``initGui()`` e ``unload()``. Come primo passo controlliamo se la versione di QGIS in uso
ha tutte le funzioni necessarie. Se i nuovi menu sono disponibili, posizioneremo il plugin in uno di essi, altrimenti utilizzeremo
il menu `Plugins`::

    def initGui(self):
      # crea l'azione che inizializza la configurazione del plugin
      self.action = QAction(QIcon(":/plugins/testplug/icon.png"), "Test plugin", self.iface.mainWindow())
      self.action.setWhatsThis("Configuration for test plugin")
      self.action.setStatusTip("This is status tip")
      QObject.connect(self.action, SIGNAL("triggered()"), self.run)

      # controlla se il menu Raster è disponibile
      if hasattr(self.iface, "addPluginToRasterMenu"):
        # Menu e barra strumenti Raster disponibili
        self.iface.addRasterToolBarIcon(self.action)
        self.iface.addPluginToRasterMenu("&Test plugins", self.action)
      else:
        # Menu e barra strumenti Raster non disponibili 
        self.iface.addToolBarIcon(self.action)
        self.iface.addPluginToMenu("&Test plugins", self.action)

      # connessione al segnale renderComplete emesso in seguito alla visualizzazione dell'area mappa
      QObject.connect(self.iface.mapCanvas(), SIGNAL("renderComplete(QPainter *)"), self.renderTest)

    def unload(self):
      # controlla se il menu Raster è disponibile e rimuove i nostri pulsanti da menu a barre strumenti appropriati
      if hasattr(self.iface, "addPluginToRasterMenu"):
        self.iface.removePluginRasterMenu("&Test plugins",self.action)
        self.iface.removeRasterToolBarIcon(self.action)
      else:
        self.iface.removePluginMenu("&Test plugins",self.action)
        self.iface.removeToolBarIcon(self.action)

      # disconnessione dal segnale dell'area mappa
      QObject.disconnect(self.iface.MapCanvas(), SIGNAL("renderComplete(QPainter *)"), self.renderTest)

L'elenco completo dei metodi è disponibile in `API docs <http://qgis.org/api/classQgisInterface.html>`_.
Le uniche funzioni che **devono** esistere sono ``initGui()`` e ``unload()``.

.. index:: plugin; file risorse, resources.qrc

File risorse
^^^^^^^^^^^^

Nel codice citato in ``initGui()`` si è usata un'icona dal file di risorse :file:`resources.qrc`::

  <RCC>
    <qresource prefix="/plugins/testplug" >
       <file>icon.png</file>
    </qresource>
  </RCC>

E' importante avere un prefisso che non collida con altri plugin o altre parti di QGIS, altrimenti si potrebbero ottenere
risorse di cui non si ha bisogno. 
Il comando :command:`pyrcc4` permette di creare un file pyuthon contenente le risorse::

  pyrcc4 -o resources.py resources.qrc

Se tutti i passi sono stati eseguiti attentamente, ora si dovrebbe essere in grado di caricare il plugin dal gestore dei plugin e di
ottenere un messaggio in console nel momento i cui si selezioni la voce di menu o il pulsante appropriati.

Nella pratica reale, conviene scrivere il plugin in una cartella dedicata e creare un "makefile" per la generazione
della UI e dei file risorse.

.. index:: plugin; documentazione, plugin; implementare una guida

Documentazione
--------------

*Questo metodo di documentazione richiede QGIS in versione minima 1.5.*

La documentazione di un plugin può essere scritta in file HTML. Il modulo
:mod:`qgis.utils` mette a disposizione la funzione :func:`showPluginHelp`, 
che apre il file di *help* allos tesso modo di qualsiasi altro *help* di QGIS.

La funzione :func:`showPluginHelp`` cerca i file di *help* nella stessa directory del metodo che la chiama.
Cercherà i file :file:`index-ll_cc.html`, :file:`index-ll.html`, :file:`index-en.html`, :file:`index-en_us.html` e
:file:`index.html`, visualizzando il primo che incontra.
 ``ll_cc`` rappresenta il *locale* di QGIS: è possibile gestire traduzioni multiple della documentazione.

La funzione :func:`showPluginHelp` può anche prendere come parametri **packageName**, 
plugin di cui visualizzare la documentazione, **"filename**, che può sostituire "index" nel nome dei file sopracitati,
e **section**, che rappresenta il nome di un tag HTML *anchor* su cui sarà posizionato il browser.

.. index:: plugin; esempi di codice

Esempi di codice
----------------

Questa sezione elenca esempi di codice per facilitare lo sviluppo di un plugin.

.. index:: plugin; chiamare metodi con scorciatoie

Chiamare un metodo con una scorcaitoia da tastiera
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Aggiungere ad ``initGui()``::

  self.keyAction = QAction("Test Plugin", self.iface.mainWindow())
  self.iface.registerMainWindowAction(self.keyAction, "F7") # l'azione è attivata dal tasto F7
  self.iface.addPluginToMenu("&Test plugins", self.keyAction)
  QObject.connect(self.keyAction, SIGNAL("triggered()"),self.keyActionF7)

Aggiungere ad ``unload()``::

  self.iface.unregisterMainWindowAction(self.keyAction)

Il metodo che viene chiamato in seguito alla pressione di F7::

  def keyActionF7(self):
    QMessageBox.information(self.iface.mainWindow(),"Ok", "Hai premuto F7")

.. index:: plugins; toggle layers

Interagire con i layer (soluzione)
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

*Nota:* a partire da QGIS 1.5, la classe :class:`QgsLegendInterface` permette una certa 
interazione con l'elenco dei layer in legenda.

Attualmente non esiste un metodo per accedere direttamente ai layer in legenda. 
Quella che segue è una soluzione che permette di accendere/spegnere un layer usando la sua trasparenza::

  def toggleLayer(self, lyrNr):
    lyr = self.iface.mapCanvas().layer(lyrNr)
    if lyr:
      cTran = lyr.getTransparency()
      lyr.setTransparency(0 if cTran > 100 else 255)
      self.iface.mapCanvas().refresh()	

Il metodo richiede il nmero del layer (0 rappresenta il primo layer in alto) e può essere chiamato da::

  self.toggleLayer(3)

.. index:: plugin; accedere agli attributi degli elementi selezionati

Accedere alla tabella degli attributi degli elementi selezionati
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
::

  def changeValue(self, value):
    layer = self.iface.activeLayer()
    if(layer):		
      nF = layer.selectedFeatureCount()
      if (nF > 0):		
      layer.startEditing()
      ob = layer.selectedFeaturesIds()
      b = QVariant(value)
      if (nF > 1):
        for i in ob:
        layer.changeAttributeValue(int(i),1,b) # 1 è la seconda colonna
      else:
        layer.changeAttributeValue(int(ob[0]),1,b) # 1 è la seconda colonna
      layer.commitChanges()
      else:
        QMessageBox.critical(self.iface.mainWindow(),"Errore", "Selezionare almeno un elemento dal layer corrente")
    else:
      QMessageBox.critical(self.iface.mainWindow(),"Errore","Selezionare un layer")
  
Il metodo richiede un parametro (nuovo valore per il campo attributo dell'elemento selezionato) e può essere chiamata da::

  self.changeValue(50)

.. index:: plugin; debugging con PDB, debugging plugin

Debug di un plugin con PDB
^^^^^^^^^^^^^^^^^^^^^^^^^^

Aggiungere il seguente codice nella zona che si intende controllare::

 # Usa pdb per il debugging
 import pdb
 # Le linee seguenti permettono di inserire un punto di interruzione nell'aplicazione
 pyqtRemoveInputHook()
 pdb.set_trace()

Quindi lanciare QGIS dalla linea di comando:

   * Linux: :command:`$ ./Qgis`
   * Mac OS X: :command:`$ /Applications/Qgis.app/Contents/MacOS/Qgis`

Quando l'applicazione raggiunge il punto di interruzione è possible usare la console.

.. index:: plugin; rilascio

Rilascio del plugin
-------------------

E' possibile pubblicare il propri plugin su `PyQGIS plugin repository <http://plugins.qgis.org/>`_.
Al link citato sono anche disponibili delle linee guida su come preparare il plugin affinchè esso 
possa funzionare con l'installatore di plugin.
Nel caso si volesse gestire in proprio la pubblicazione, creare un file XML contenente l'elenco dei plugin ed i loro metadati:
si vedano degli esempi in `plugin repositories <http://www.qgis.org/wiki/Python_Plugin_Repositories>`_.

.. index:: plugin; configurare IDE Windows

Configurare la IDE su Windows
-----------------------------

Su Linux non è necessaria alcuna configurazione aggiuntiva per sviluppare plugin.
In Windows, invece, bisogna assicurarsi di avere le stesse impostazioni d'ambiente
ed usare le stesse librerie ed interpreti di QGIS. A tale scopo, bisogna modificare il 
file batch di avvio di QGIS. Se si è installato QGIS con OSGeo4W, il file potrebbe trovarsi 
al seguente percorso: :file:`C:\\OSGeo4W\\bin\\qgis-unstable.bat`.

Di seguito si mostra come impostare `Pyscripter IDE <http://code.google.com/p/pyscripter>`_: 
altre IDE potrebbero richiedere approcci leggermente differenti:

* Creare una copia di :file:`qgis-unstable.bat` e rinominarla in :file:`pyscripter.bat`
* Aprire  :file:`pyscripter.bat` con un editor di testo ed eliminare l'ultima riga, quella che avvia QGIS
* Aggiungere una linea che punta all'eseguibile di *pyscripter* ed aggiungere l'argomento da linea di comando che imposta la versione di python da usare:
  python 2.5 in QGIS 1.3
* Aggiungere l'argomento che punta alla cartella contenente le *dll* usate da QGIS::

    @echo off
    SET OSGEO4W_ROOT=C:\OSGeo4W
    call "%OSGEO4W_ROOT%"\bin\o4w_env.bat
    call "%OSGEO4W_ROOT%"\bin\gdal16.bat
    @echo off
    path %PATH%;%GISBASE%\bin
    Start C:\pyscripter\pyscripter.exe --python25 --pythondllpath=C:\OSGeo4W\bin
