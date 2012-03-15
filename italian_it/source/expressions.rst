.. index:: espressioni, filtri, calcolo valori

.. _expressions:

Espressioni, filtri e calcolo valori
====================================

QGIS supporta il parsing di espressioni SQL-like; è supportato solo un sottoinsieme minimo di sintassi SQL.
Le espressioni possono essere valutate come predicati booleani (che restituiscono Vero/Falso) o come funzioni (che restituiscono valori scalari).

Sono supportati tre tipi di base:

* numeri --- numeri interi e decimali, es. ``123``, ``3.14``
* stringhe --- devono essere chiuse tra apici singoli: ``'hello world'``
* riferimento colonna --- quando valutato, il riferimento è sostituito dal valore effettivo del campo. Non c'è "escape" dei nomi.

Sono disponibili le seguenti operazioni:

* operatori aritmetici: ``+``, ``-``, ``*``, ``/``, ``^``
* parentesi: per forzare la precedenza degli operatori: ``(1 + 1) * 3``
* più e meno unari: ``-12``, ``+5``
* funzioni matematiche: ``sqrt``, ``sin``, ``cos``, ``tan``, ``asin``, ``acos``, ``atan``
* funzioni geometriche: ``$area``, ``$length``
* funzioni di conversione: ``to int``, ``to real``, ``to string``

Sono disponibili i seguenti predicati:

* confronto: ``=``, ``!=``, ``>``, ``>=``, ``<``, ``<=``
* confronto di pattern: ``LIKE`` (usando % e _), ``~`` (espressioni regolari)
* predicati logici: ``AND``, ``OR``, ``NOT``
* controllo valori NULL: ``IS NULL``, ``IS NOT NULL``

**Compatibilità:** le funzioni matematiche, geometriche, di conversione e l'operatore potenza ``^`` sono disponibili a partire 
dalla versione 1.4 di QGIS.

Esempi di predicati:

* ``1 + 2 = 3``
* ``sin(angle) > 0``
* ``'Hello' LIKE 'He%'``
* ``(x > 10 AND y > 10) OR z = 0``

Esempi di espressioni scalari:

* ``2 ^ 10``
* ``sqrt(val)``
* ``$length + 1``

.. index:: espressioni; parsing

Parsing di espressioni
----------------------

**TODO:** parsing, error handling

::

  >>> s = QgsSearchString()
  >>> s.setString("1 + 1 = 2")
  True
  >>> s.setString("1 + 1 =")
  False
  >>> s.parserErrorMsg()
  PyQt4.QtCore.QString(u'syntax error, unexpected $end')

**TODO:** working with the tree, evaluation as a predicate, as a function, error handling

.. index:: espressioni; valutazione

Valutazione di espressioni
--------------------------

::

  st = ss.tree()
  if not st:
    raise ValueError, "empty expression was used"

  print st.makeSearchString()

  res = st.checkAgainst(fields, feature.attributeMap())

  res, value = st.getValue(st, fields, feature.attributeMap(), feature.geometry())

  print st.errorMsg()
