 .. index:: composizione mappa, stampa mappa

.. _composer:

Composizione e stampa della mappa
=================================

Ci sono due approcci per la rappresentazione in mappa dei dati geografici: il modo più veloce consiste nell'usare :class:`QgsMapRenderer`,
per avere, invece, un controllo maggiore sull'output si può usare la classe :class:`QgsComposition`.

.. index:: rappresentazione mappa; semplice

Rappresentazione semplice
-------------------------

Esempio di rappresentazione usando :class:`QgsMapRenderer`::

  # crea immagine
  img = QImage(QSize(800,600), QImage.Format_ARGB32_Premultiplied)

  # imposta colore di sfondo
  color = QColor(255,255,255)
  img.fill(color.rgb())

  # crea painter
  p = QPainter()
  p.begin(img)
  p.setRenderHint(QPainter.Antialiasing)

  render = QgsMapRender()

  # imposta insieme di layer
  lst = [ layer.getLayerID() ]  # add ID of every layer
  render.setLayerSet(lst)

  # imposta estensione
  rect = QgsRect(render.fullExtent())
  rect.scale(1.1)
  render.setExtent(rect)

  # imposta dimensione output
  render.setOutputSize(img.size(), img.logicalDpiX())

  # rendering
  render.render(p)

  p.end()

  # .. .. salva immagine
  img.save("render.png","png")

.. index:: output; usare il compositore di stampe

Utilizzare il compositore di stampe
-----------------------------------

Il sistema di stampa è uno strumento maneggevole che permette di creare un output più completo rispetto al metodo appena mostrato.
E' possibile creare impaginazioni complesse contenenti mappe, etichette, legende, tabelle e tutti gli elementi che sono solitamente
presenti in una mappa cartacea. Le impaginazioni possono essere esportate in formato pdf, in formato immagine o direttamente stampate.

Il sistema di stampa consiste di una serie di classi appartenenti alla libreria core. QGIS ha una GUI dedicata al posizionamento
degli elementi di mappa, che però non è disponibile nella libreria gui.
Se non si è familiari con il framework `Qt Graphics View <http://doc.qt.nokia.com/stable/graphicsview.html>`_, conviene dargli 
uno sguardo ora in quanto il sistema di stampa si basa su di esso.

La classe principale è :class:`QgsComposition`, derivata dalla classe :class:`QGraphicsScene`. Creiamone una::

  mapRenderer = iface.mapCanvas().mapRenderer()
  c = QgsComposition(mapRenderer)
  c.setPlotStyle(QgsComposition.Print)

Si noti che la composizione prende un'istanza della classe :class:`QgsMapRenderer`. Supponiamo di eseguire il codice all'interno
di QGIS, per cui usiamo il visualizzatore dall'area mappa. La composizione usa vari parametri, come l'insieme di default dei layer di mappa
e l'estensione corrente. Quando si usa il sistema di stampa in un'applicazione standalone, si può creare l'istanza del proprio visualizzatore
come mostrato precedentemente e passarlo alla composizione.

E' possibile aggiungere vari elementi (mappa, etichette, ...) - questi elementi discendono dalla classe :class:`QgsComposerItem`.
Gli elementi supportati sono:

* mappa - questo elemento dice alla libreria dove posizione la mappa stessa. Nell'esempio creiamo una mappa e la estendiamo alla dimensione della carta::
  
    x, y = 0, 0
    w, h = c.paperWidth(), c.paperHeight()
    composerMap = QgsComposerMap(c, x,y,w,h)
    c.addItem(composerMap)

* etichettal - permette di visualizzare delle etichette. E' possibile modificare font, colore, allineamento e margini.
  ::

    composerLabel = QgsComposerLabel(c)
    composerLabel.setText("Hello world")
    composerLabel.adjustSizeToText()
    c.addItem(composerLabel)

* legenda
  ::

    legend = QgsComposerLegend(c)
    legend.model().setLayerSet(mapRenderer.layerSet())
    c.addItem(legend)

* barra della scala
  ::

    item = QgsComposerScaleBar(c)
    item.setStyle('Numeric') # optionally modify the style
    item.setComposerMap(composerMap)
    item.applyDefaultSize()
    c.addItem(item)

* freccia
* immagine
* forma
* tabella

Di default, gli elementi creati sono posizionati nel punto in alto a sinistra della pagina ed hanno dimensione pari a zero.
Posizione e dimensione sono misurate in millimetri::

  # imposta la posizione dell'etichetta ad 1cm dall'alto ed a 2cm da sinistra della pagina
  composerLabel.setItemPosition(20,10)
  # imposta posizione e dimensione dell'etichetta (larghezza 10cm, altezza 3cm)
  composerLabel.setItemPosition(20,10, 100, 30)

Intorno ad ogni elemento viene disegnata una cornire: per rimuoverla::

  composerLabel.setFrame(False)

QGIS mette a disposizione anche degli schemi predefiniti di composizione salvati in file .qpt file (con una sintassi XML).
Purtroppo tale possibilità non è implementata nella API.

Una volta la composizione pronta, possiamo crearne l'output in formato vettoriale o raster.

Le impostazioni di default dell'output sono: dimensione della pagina A4, risoluzione 300 DPI. Se necessario è possible modificare tali impostazioni.
Le dimensioni della pagina sono specificate in millimetri::

  c.setPaperSize(width, height)
  c.setPrintResolution(dpi)

.. index:: output; immagine raster

Output come immagine raster
~~~~~~~~~~~~~~~~~~~~~~~~~~~

Il codice seguente mostra come creare un output in formato immagine::

  dpi = c.printResolution()
  dpmm = dpi / 25.4
  width = int(dpmm * c.paperWidth())
  height = int(dpmm * c.paperHeight())

  # crea l'iimagine di output e l'inizializza
  image = QImage(QSize(width, height), QImage.Format_ARGB32)
  image.setDotsPerMeterX(dpmm * 1000)
  image.setDotsPerMeterY(dpmm * 1000)
  image.fill(0)

  # rendering della composizione
  imagePainter = QPainter(image)
  sourceArea = QRectF(0, 0, c.paperWidth(), c.paperHeight())
  targetArea = QRectF(0, 0, width, height)
  c.render(imagePainter, targetArea, sourceArea)
  imagePainter.end()

  image.save("out.png", "png")

.. index:: output; PDF

Output come PDF
~~~~~~~~~~~~~~~

Il codice seguente mostra come salvare l'output in un file PDF::

  printer = QPrinter()
  printer.setOutputFormat(QPrinter.PdfFormat)
  printer.setOutputFileName("out.pdf")
  printer.setPaperSize(QSizeF(c.paperWidth(), c.paperHeight()), QPrinter.Millimeter)
  printer.setFullPage(True)
  printer.setColorMode(QPrinter.Color)
  printer.setResolution(c.printResolution())
  
  pdfPainter = QPainter(printer)
  paperRectMM = printer.pageRect(QPrinter.Millimeter)
  paperRectPixel = printer.pageRect(QPrinter.DevicePixel)
  c.render(pdfPainter, paperRectPixel, paperRectMM)
  pdfPainter.end()
