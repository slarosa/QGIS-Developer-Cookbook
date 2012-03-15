.. index:: misure

.. _measure:

Misure
======

Per misurare distanze o aree usare la classe :class:`QgsDistanceArea`. Se la proiezione non è abilitata, il calcolo sarà planare, altrimenti sarà riferito
ad un ellissoide. Quando l'ellossoide non è impostato, di default viene usato WGS84. 

::

  d = QgsDistanceArea()
  d.setProjectionsEnabled(True)
  
  print "distance in meters: ", d.measureLine(QgsPoint(10,10),QgsPoint(11,11))

**TODO:** area, planar vs. ellipsoid
