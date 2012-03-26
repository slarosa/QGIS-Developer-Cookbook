.. index:: geometria; gestione

.. _geometry:

Gestire la geometria
====================

Punti, linee e poligoni che rappresentano elementi spaziali sono comunemente detti geometrie. 
In QGIS sono rappresentati dalla classe :class:`QgsGeometry`.
Tutti i tipi di geometrie possibili sono descritti in `JTS discussion page <http://www.vividsolutions.com/jts/discussion.htm#spatialDataModel>`_.

Una geometria può essere, in realtà, anche data da una collezione di geometrie semplici (dette a parte singola); tale tipo di geometria è detto "geometria multi-parte". Se contiene un solo tipo di geometria semplice, la chiamiamo multi-punto, multi-linea, multi-poligono.
Ad esempio, uno stato che consiste di più isole può essere rappresentato come un multi-poligono.

Le coordinate delle geometrie possono essere espresse in qualsiasi sistema di riferimento (CRS). Quando di recuperano elementi da un layer, le geometrie associate hanno coordinate espresse nel sistema di riferimento del layer stesso.

.. index:: geometria; costruzione

Costruire una geometria
-----------------------

Ci sono diverse opzioni per la creazione di una geometria:

* da coordinate::

    gPnt = QgsGeometry.fromPoint(QgsPoint(1,1))
    gLine = QgsGeometry.fromPolyline( [ QgsPoint(1,1), QgsPoint(2,2) ] )
    gPolygon = QgsGeometry.fromPolygon( [ [ QgsPoint(1,1), QgsPoint(2,2), QgsPoint(2,1) ] ] )

  Le coordinate sono date usando la classe :class:`QgsPoint`. 

  Una polilinea è rappresentata da una lista di punti. Un poligono è rappresentato da una lista di linee che di richiudono; un poligono può contenere dei buchi.

  Un multi-poligono è una lista di punti. Una multi-linea è una lista di linee. Un multi-poligono è una lista di poligoni.

* da well-known text (WKT)::

    gem = QgsGeometry.fromWkt("POINT (3 4)")

* da well-known binary (WKB)::

    g = QgsGeometry()
    g.setWkbAndOwnership(wkb, len(wkb))


.. index:: geometria; accesso

Accedere ad una geometria
-------------------------

Per accedere ad una geometria bisogna come prima cosa conoscere il tipo di geometria. Il medoto :func:`wkbType` restituisce un valore dall'enumerazione QGis.WkbType enumeration::

  >>> gPnt.wkbType() == QGis.WKBPoint
  True
  >>> gLine.wkbType() == QGis.WKBLineString
  True
  >>> gPolygon.wkbType() == QGis.WKBPolygon
  True
  >>> gPolygon.wkbType() == QGis.WKBMultiPolygon
  False

In alternativa è possibile usare il metodo :func:`type`, che restituisce una valore dall'enumerazione QGis.GeometryType.
La funzione :func:`isMultipart` ci dice se una geometria è multi-parte oppure no.

Per estrarre informazioni da una geometria ci sono delle funzioni di accesso per ogni tipo di vettore::

  >>> gPnt.asPoint()
  (1,1)
  >>> gLine.asPolyline()
  [(1,1), (2,2)]
  >>> gPolygon.asPolygon()
  [[(1,1), (2,2), (2,1), (1,1)]]

.. note:: le tuple (x,y) non sono tuple reali, ma oggetti di :class:`QgsPoint`, i cui valori sono accessibili con i metodi :func:`x` e :func:`y`.

Per le geometrie multiparte ci sono funzioni simili: :func:`asMultiPoint`, :func:`asMultiPolyline`, :func:`asMultiPolygon()`.

.. index:: geometria; predicati ed operazioni

Predicati ed operazioni 
-----------------------

QGIS usa la libreria GEOS per operazioni avanzate sulla geometria, come i predicati (:func:`contains`, :func:`intersects`, ...)
e le operazioni (:func:`union`, :func:`difference`, ...).

**TODO:**

* :func:`area`, :func:`length`, :func:`distance`
* :func:`transform`
* available predicates and set operations
