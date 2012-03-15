.. index:: impostazioni; leggere, impostazioni; memorizzare

.. settings:

Leggere e memorizzare le impostazioni
=====================================

E' spesso utile, nel caso di sviluppo di plugin, memorizzare alcune variabili in modo tale che l'utente non è obbligato ad inserirle o selezionarle successivamente.

Le variabili possono essere salvate tramite le Qt e le API di QGIS. Per ogni variabile deve essere definita una chiave che ne faciliti l'accesso - es. per il colore favorito dell'utente si potrebbe usare la chiave "colore_favorito" o altre stringhe significative. Si raccomanda di strutturare in qualche modo l'associazione di nomi alle chiavi.

Vi sono diversi tipi di impostazioni:

.. index:: impostazione; globali

* **impostazioni globali** - sono specifiche del computer su cui si sta lavorando. QGIS stesso memorizza diverse impostazioni globali,
  come ad esempio la dimensione della finestra principale o la tolleranza di snapping predefinita. Di tale funzionalità si occupa direttamente il framework
  Qt attraverso la classe :class:`QSettings`.
  Di default, questa classe memorizza le impostazioni nella modalità nativa del sistema in uso - registro (su Windows), file .plist (su Mac OS X), file .ini (su Unix).
  Ulteriori dettagli nella `documentazione QSettings <http://doc.qt.nokia.com/stable/qsettings.html>`_. Di seguito solo alcuni esempi::

    def store():
      s = QSettings()
      s.setValue("myplugin/mytext", "hello world")
      s.setValue("myplugin/myint",  10)
      s.setValue("myplugin/myreal", 3.14)

    def read():
      s = QSettings()
      mytext = s.value("myplugin/mytext", "default text").toString()
      myint  = s.value("myplugin/myint", 123).toInt()[0]
      myreal = s.value("myplugin/myreal", 2.71).toDouble()[0]
  
  Nei metodi setValue() e value() Qt usa instanze di QVariant per i valori delle variabili. I valori sono automaticamente convertiti da Python a
  QVariant; la conversione da QVariant a Python, invece, non è automatica, per cui si usano i metodi to*(). Ancora qualche dettaglio:

  * il secondo parametro del metodo value() è opzionale e serve a definire il valore predefinito in assenza di una precedente impostazione del valore stesso
  * toString() restituisce un'istanza QString, non una stringa Python
  * toInt() e toDouble() restituiscono tuple (valore, ok) - il secondo elemento indica True se la conversione da QVariant è andata a buon fine.
  
.. index:: impostazioni; progetto

* **impostazioni progetto** - variano da progetto a progetto e sono connesse ad un file progetto.
  Ne sono un esempio lo sfondo dell'area mappa ed il sistema di riferimento - bianco e WGS84 potrebbero andar bene per un progetto, giallo e UTM per un 
  altro::

    proj = QgsProject.instance()

    # memorizza i valori
    proj.writeEntry("myplugin", "mytext", "hello world")
    proj.writeEntry("myplugin", "myint", 10)

    # legge i valori
    mytext = proj.readEntry("myplugin", "mytext", "default text")[0]
    myint = proj.readNumEntry("myplugin", "myint", 123)[0]

  La classe :class:`QgsProject` sarà aggiornata per fornire delle API simili a quelle della classe :class:`QSettings`. A causa di limiti del
  binding a Python, non è possibile salvare numeri a virgola mobile.

.. index:: impostazioni; layer

* **impostazioni layer di mappa** - sono impostazioni relative ad una particolare istanza di un layer di mappa all'interno di un progetto. 
  *Non* sono connesse con la fonte dati del layer, per cui se si creano due istanze layer dallo stesso shapefile le loro impostazioni *non*
  saranno condivise. Queste impostazioni sono salvate nel file di progetto. L'API è simile a  QSettings - prende e restituisce istanze di QVariant::

   # salva un value
   layer.setCustomProperty("mytext", "hello world")

   # legge il valore
   mytext = layer.customProperty("mytext", "default text").toString()

**TODO:**
   Keys for settings that can be shared among plugins
