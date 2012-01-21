.. index:: измерения

.. _measure:

Измерения
=========

Для вычисления расстояний или площадей используется класс :class:`QgsDistanceArea`.
Если перепроецирование выключено, расчёт будет выполняться на плоскости,
в противном случае --- на эллипсоиде. Когда эллипсоид не задан, для расчётов
используются параметры WGS84. ::

  d = QgsDistanceArea()
  d.setProjectionsEnabled(True)

  print "distance in meters: ", d.measureLine(QgsPoint(10,10),QgsPoint(11,11))


**TODO:** area, planar vs. ellipsoid
