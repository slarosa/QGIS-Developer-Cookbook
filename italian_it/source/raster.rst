
.. _raster:

.. index::layer raster; utilizzare

Utilizzare layer Raster
=======================

Questa sezione elenca le varie operazioni possibili con i dati raster.

.. index:: layer raster; dettagli

Dettagli del layer
------------------

Un layer raster consiste di una o più bande raster - si parla di raster a banda singola o multi-banda.
Una banda rappresenta una matrice di valori. Le immagini a colori (es. le foto aeree) sono raster composti di una banda
rossa, una blu ed una verde.
I raster a banda singola possono rappresentare variabili continue (es. l'elevazione) o variabili discrete (es. uso del suolo).
In alcuni casi, un layer raster ha associata una tavolozza (palette) ed i valori del raster corrispondono a colori memorizzati nella tavolozza. 

  >>> rlayer.width(), rlayer.height()
  (812, 301)
  >>> rlayer.extent().toString()
  PyQt4.QtCore.QString(u'12.095833,48.552777 : 18.863888,51.056944')
  >>> rlayer.rasterType()
  2  # 0 = GrayOrUndefined (single band), 1 = Palette (single band), 2 = Multiband
  >>> rlayer.bandCount()
  3
  >>> rlayer.metadata()
  PyQt4.QtCore.QString(u'<p class="glossy">Driver:</p>...')
  >>> rlayer.hasPyramids()
  False

.. index:: raster layers; drawing style

Stile di disegno
----------------

Quando un layer raster viene caricato, esso ha associato uno stile di disegno di default. Lo stile può essere modificato tramite le proprietà del layer o via programmazione. 
Sono disponibili i seguenti stili di disegno:

====== =============================== ===============================================================================================
Indice   Costante: QgsRasterLater.X     Commento
====== =============================== ===============================================================================================
  1     SingleBandGray                 Banda singola in gradazione di grigi
  2     SingleBandPseudoColor          Banda singola in pseudocolore
  3     PalettedColor                  Immagine "palette" con tabella di colori
  4     PalettedSingleBandGray         Immagine "palette" in gradazione di grigi
  5     PalettedSingleBandPseudoColor  Immagine "palette" in pseudocolore
  7     MultiBandSingleBandGray        Layer con due o più bande; una sola banda rappresentata in gradazione di grigi
  8     MultiBandSingleBandPseudoColor Layer con due o più bande; una sola banda rappresentata in pseudocolore
  9     MultiBandColor                 Layer con due o più bande rappresentate in RGB
====== =============================== ===============================================================================================

Per conoscere lo stile corrente:

  >>> rlayer.drawingStyle()
  9

I layer a banda singola possono essere rappresentati in gradazioni di grigi (valori bassi = nero, valori alti = bianco) o con un algoritmo pseudocolore, che assegna dei colori ai valori della banda singola. I raster a banda singola e con tavolozza possono essere rappresentati utilizzando la tavolozza stessa.
I layer multi-banda sono rappresentati mappando le bande a colori RGB: è anche possibile rappresentare una sola banda in gradazioni di grigi o in pseudocolore.

La sezione seguente spiega come interrogare e modificare lo stile di disegno di un layer. Una volta fatti i cambiamenti del caso,
bisogna aggiornare la vista mappa (vedi :ref:`refresh-layer`).

**TODO:** contrast enhancements, transparency (no data), user defined min/max, band statistics

.. index:: raster; banda singola

Raster a banda singola
----------------------

I raster a banda singola sono rappresentati di default in gradazioni di grigi. Per cambiare lo stile in pseudocolore:

  >>> rlayer.setDrawingStyle(QgsRasterLayer.SingleBandPseudoColor)
  >>> rlayer.setColorShadingAlgorithm(QgsRasterLayer.PseudoColorShader)

Il metodo ``PseudoColorShader`` evidenzia i valori bassi in blu ed i valori alti in rosso. Il metodo ``FreakOutShader``
usa colori più fantasiosi. 
Il metodo ``ColorRampShader`` mappa i colori come specificato nella sua mappa colore. Il metodo ha tre modalità di interpolazione dei valori:

* lineare (``INTERPOLATED``): il colore risultante è un'interpolazione lineare delle voci di colore sopra e sotto il valore del pixel corrente
* discreto (``DISCRETE``): è usato il colore della voce della mappa colore con valore uguale o maggiore
* esatto (``EXACT``): il colore non è interpolato. Solo i pixel con valori uguali alle voci della mappa colore vengono disegnati

Per impostare una rampa colore interpolata dal verde al giallo (pere valori di pixel da 0 a 255)::

  >>> rlayer.setColorShadingAlgorithm(QgsRasterLayer.ColorRampShader)
  >>> lst = [ QgsColorRampShader.ColorRampItem(0, QColor(0,255,0)), QgsColorRampShader.ColorRampItem(255, QColor(255,255,0)) ]
  >>> fcn = rlayer.rasterShader().rasterShaderFunction()
  >>> fcn.setColorRampType(QgsColorRampShader.INTERPOLATED)
  >>> fcn.setColorRampItemList(lst)

Per ritornare ai livelli di grigio di default:

  >>> rlayer.setDrawingStyle(QgsRasterLayer.SingleBandGray)

.. index:: raster; multi-banda

Raster multi-banda
------------------

Di default QGIS mappa le prime tre bande con i colori rosso, verde e blu in modo da creare un'immagine a colori (stile ``MultiBandColor``).
E' possibile sovrascrivere tali impostazioni. Il codice seguente scambia la banda rossa (1) con la verde (2):

  >>> rlayer.setGreenBandName(rlayer.bandName(1))
  >>> rlayer.setRedBandName(rlayer.bandName(2))

Se si visualizza una sola banda, è possibile utilizzare gli stili per banda singola - gradazione di grigi o pseudocolore::

  >>> rlayer.setDrawingStyle(QgsRasterLayer.MultiBandSingleBandPseudoColor)
  >>> rlayer.setGrayBandName(rlayer.bandName(1))
  >>> rlayer.setColorShadingAlgorithm(QgsRasterLayer.PseudoColorShader)
  >>> # now set the shader

.. index:: 
  pair: layer raster; aggiornare

.. _refresh-layer:

Aggiornare i layer
------------------

Per rendere subito disponibile all'utente i cambiamenti alla simbologia di un layer, chiamare i seguenti metodi::

   if hasattr(layer, "setCacheImage"): layer.setCacheImage(None)
   layer.triggerRepaint()

La prima chiamata cancella la cache dell'immagine, nel caso in cui il caching del visualizzatore sia attivato. Tale funzionalità è disponibile a partire dalla versione 1.4 di QGIS: per assicurarsi che il codice funzioni con tutte le versioni di QGIS, si verifica prima se il metodo esiste.
La seconda chiamata forza un aggiornamento della vista mappa.

Per forzare QGIS ad aggiornare la simbologia del layer anche in legenda, usare il seguente codice (``iface`` è un'istanza di QgisInterface)::

   iface.legendInterface().refreshLayerSymbology(layer)

.. index::
  pair: layer raster; query

Query sui valori
----------------

Per effettuare una query sui valori delle bande di una layer raster in un punto specifico::

  res, ident = rlayer.identify(QgsPoint(15.30,40.98))
  for (k,v) in ident.iteritems():
    print str(k),":",str(v)

La funzione "identify" restituisce Vero/Falso ed un dizionario: le chiavi sono i nomi delle bande, i valori sono relativi al punto scelto.
Chiave e valore sono instanze di QString: per vedere il valore effettivo, vanno convertiti in stringhe python.
