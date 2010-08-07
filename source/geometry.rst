
.. _geometry:

Geometry Handling
=================

Points, linestrings, polygons (and more complex shapes) are commonly referred to as geometries. They are represented using :class:`QgsGeometry` class.

To extract information from geometry there are accessor functions for every vector type. How do accessors work:

:func:`asPoint`
  returns :class:`QgsPoint`
:func:`asPolyline`
  returns a list of :class:`QgsPoint` items
:func:`asPolygon` 
  returns a list of rings, every ring consists of a list of :class:`QgsPoint` items. First ring is outer, subsequent rings are holes.



You can use :func:`QgsGeometry.isMultipart` to find out whether the feature is multipart. For multipart features there are similar accessor functions:
:func:`asMultiPoint`, :func:`asMultiPolyline`, :func:`asMultiPolygon()`.

There are several options how to create a geometry:

* from coordinates::

    gPnt = QgsGeometry.fromPoint(QgsPoint(1,1))
    gLine = QgsGeometry.fromPolyline( [ QgsPoint(1,1), QgsPoint(2,2) ] )
    gPolygon = QgsGeometry.fromPolygon( [ [ QgsPoint(1,1), QgsPoint(2,2), QgsPoint(2,1) ] ] )

* from well-known text (WKT)::

    gem = QgsGeometry.fromWkt("POINT (3 4)")

* from well-known binary (WKB)::

    g = QgsGeometry()
    g.setWkbAndOwnership(wkb, len(wkb))

