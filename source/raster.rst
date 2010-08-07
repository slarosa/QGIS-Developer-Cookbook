
.. _raster:

Using Raster Layers
===================

This sections lists various operations you can do with raster layers.

Query Values
------------

To do a query on value of bands of raster layer at some specified point::

  ident = rlayer.identify(QgsPoint(15.30,40.98))
  for (k,v) in ident.iteritems():
    print str(k),":",str(v)

The identify function returns a dictionary - keys are band names, values are the values at chosen point.
Both key and value are QString instances so to see actual value you'll need to convert them to python strings (as shown in code snippet). 

