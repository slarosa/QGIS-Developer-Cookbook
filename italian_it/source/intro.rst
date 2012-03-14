
.. _introduction:

==============
 Introduzione
==============

Questo documento intende essere sia un tutorial che una guida di riferimento
e fornisce una buona panoramica delle funzionalità principali di PyQGIS, nonostante non
elenchi tutti i possibili casi d'uso.

A partire dalla release 0.9, QGIS un supporto opzionale allo scripting con
Python, uno dei linguaggi di scripting più utilizzati. Il binding a PyQGIS
dipende da SIP e PyQT4. La ragione dell'utilizzo di SIP, al posto di SWIG,
dipende dal fatto che il codice di QGIS dipende dalle librerie Qt. Il binding 
a Python per le Qt (PyQT) utilizza SIP. il che permette una facile integrazione
di PyQGIS con PyQt. 

**TODO:**
   Getting PyQGIS to work (Manual compilation, Troubleshooting)

Ci sono diversi modi per utilizzare il binding a Python di QGIS:  

* inserire comandi nella console Python di QGIS
* creare ed usare plugin in Python
* creare applicazioni personalizzate con le API di QGIS

.. index:: API

Le API di QGIS sono estensivamente documentate al seguente `link <http://www.qgis.org/api/>`_.
Le API QGIS in PYTHON sono praticamente identiche alle API in C++.
 
Nel `blog di QGIS <http://blog.qgis.org/>`_ ci sono varie risorse sulla programmazione
con PyQGIS. Si guardi il `tutorial QGIS e Python <http://blog.qgis.org/?q=node/59>`_
per alcuni esempi ed applicazioni. Se si intende sviluppare plugin, si suggerisce di
scaricarne alcuni dal `repository dei plugin <http://plugins.qgis.org/>`_ e studiarne il codice.

.. index::
  pair: Python; console

La console Python
=================

La console Python può essere aperta dal menu di QGIS: :menuselection:`Plugins --> Console python`.
La console si pare in una nuova finestra:

.. image:: console.png

La figura mostra come ottenere informazioni, come l'ID ed in numero di entità, 
sul layer selezionato nella TOC di QGIS.
Per interagire con l'ambiente QGIS bisogna utilizzare la variabile :data:`qgis.utils.iface`, 
che è un'istanza della classe :class:`QgisInterface`. Quest'interfaccia permette di interagire con
la vista mappa, i menu, le barre degli strumenti ed altre componenti di QGIS.

Per convenienza dell'utente, i comandi seguenti sono da eseguire non appena si lancia la console
(in futuro si mostreranno altri comandi di inizializzazione)::

  from qgis.core import *
  import qgis.utils

Per chi usa spesso la console, può essere utile impostare una scorciatoia da tastiera da
:menuselection:`Impostazioni  --> Configura le scorciatoie...`

.. index:: Python; plugin

Plugin Python
=============

Le funzionalità di QGIS possono essere estese tramite l'uso dei plugin. Per le prime versioni
di QGIS lo sviluppo di plugin era possibile solo in C++, mentre ora, grazie a PyQGIS, è
possibile anche usando Python. 
Python permette uno sviluppo ed una distribuzione dei plugin più agevoli. 

Dall'introduzione del supporto a Python sono stati creati molti plugin; l'installatore
di plugin ne permette facilmente l'acquisizione, l'aggiornamento e la rimozione.
Alla pagina web `Python Plugin Repositories <http://www.qgis.org/wiki/Python_Plugin_Repositories>`_
sono disponibili svariate fonti di plugin.

La creazione dei plugin è semplice: si veda :ref:`plugins` per istruzioni dettagliate.

.. index::
  pair: Python; applicazioni personalizate

Applicazioni Python
===================

Spesso processando dati GIS risulta più efficace creare degli script per automatizzare alcune
operazioni ripetitive.
Con PyQGIS lo sviluppo di script è molto semplice, basta importare ed inizializzare il modulo :mod:`qgis.core`.

E' inoltre possibile implementare applicazioni interattive dotate di alcune delle funzionalità di QGIS. 
Il modulo :mod:`qgis.gui` permette di accedere ai componenti dell'interfaccia grafica, come ad esempio la
vista mappa, che può essere facilmente integrata nell'applicazione insieme anche ai suoi strumenti (pan, zoom, etc.).


Utilizzare PyQGIS nelle applicazioni personalizzate
---------------------------------------------------

.. note:: *non* usare :file:`qgis.py` come nome per i propri script di test; Python non sarà in grado di importare i binding.

Come prima cosa bisogna importare il modulo CORE di QGIS :py:mod:`qgis.core`. Successivamente viene impostato il percorso all'installazione di QGIS attraverso la funzione :func:`setPrefixPath` della classe :class:`QgsApplication` in modo da poter accedere a tutte le librerie e le risorse disponibile. E' necessario inizializzare la funzione :func:`initQgis` affinchè tutte le risorse vengono caricate correttamente.

::

  from qgis.core import *

  # supply path to where is your qgis installed
  QgsApplication.setPrefixPath("/path/to/qgis/installation", True)

  # load providers
  QgsApplication.initQgis()

Ora è possibile lavorare con le API di QGIS, aggiungere layer, processare i dati, lanciare una
finestra con una vista mappa. Le possibilità sono infinite :-)

Per terminare le operazioni è necessario lanciare la funzione :func:`exitQgis`::

  QgsApplication.exitQgis()

.. index::
  pair: applicazioni personalizzate; eseguire

Eseguire applicazioni personalizzate
------------------------------------

Se si è installato QGIS in un percorso personalizzato diverso da quelli standard, 
Python darà il seguente messaggio di errore::

  >>> import qgis.core
  ImportError: No module named qgis.core

Il problema può essere aggirato impostando la viariabile d'ambiente ``PYTHONPATH``.
Nei comandi che seguono, ``qgispath`` va rimpiazzato dal vostro percorso di 
installazione di QGIS:

* Linux: :command:`export PYTHONPATH=/qgispath/share/qgis/python`
* Windows: :command:`set PYTHONPATH=c:\\qgispath\\python`

Il percorso ai moduli PyQGIS non è noto e dipende dalle librerie ``qgis_core``
e ``qgis_gui`` (i moduli Python servono solo come wrapper). Il percorso a tali
librerie è sconosciuto al sistema operativo, per cui si riceverà l'errore::

  >>> import qgis.core
  ImportError: libqgis_core.so.1.5.0: cannot open shared object file: No such file or directory

La soluzione sta nell'aggiungere le directory delle librerie QGIS all percorso di 
ricerca del linker dinamico:

* Linux: :command:`export LD_LIBRARY_PATH=/qgispath/lib`
* Windows: :command:`set PATH=C:\\qgispath;%PATH%`

Questi comandi possono anche essere inseriti in uno script di startup (file batch per Windows e shell script per Linux). Quando si implementano applicazioni personalizzate tramite PyQGIS, di solito ci sono due possibilità per i modelli di sviluppo:

* richiedere all'utente di installare QGIS sul proprio sistema prima di
  installare l'applcazione personalizzata. L'installer dell'applicazione
  cerca le libreri QGIS nei percorsi standard, altrimenti deve permettere
  all'utente di impostare il percorso. Tale approccio ha il vantaggio della
  semplicità di distribuzione, ma richiede l'intervento dell'utente.

* includere QGIS nel proprio applicativo. La distribuzione è più problematica,
  ma si evita all'utente di scaricare ed installare software aggiuntivo.

I due modelli di sviluppo possono essere mixati - sviluppare applicazioni
standalone per Windows e OSX, lasciare all'utente ed al gestore di pacchetti (apt, yum etc.) l'installazione di QGIS
in Linux.
