
.. _network-analysis:

Network analysis library
========================

Starting from `ee19294562 <https://github.com/qgis/Quantum-GIS/commit/ee19294562b00c6ce957945f14c1727210cffdf7>`_
(QGIS >= 1.8) the new network analysis library was added to the QGIS core
analysis library. This library:

 * can create mathematical graph from geographical data (polyline vector layers)
 * implements basics method of the graph theory (currently only Dijkstra's
   algorithm)

Network analysis library was created by exporting basics functions from
RoadGraph core plugin and now you can use it it's methods in plugins or
directly from Python console.

General information
-------------------

In short typical use case can be described as:

1. create graph from geodata (usualy polyline vector layer)
2. run graph anlysis
3. use analysis results (for example, visualize them)

Building graph
--------------

The first thing you need to do --- is to prepare input data, that is to
convert vector layert into graph. All further actions will use this graph
not layer.

As source we can use any polyline vector layer. Nodes of the polylines
becomes a graph vertices, and segments of the polylines are graph edges.
If several  nodes have same coordinates then they are same graph vertex.
So two lines that have a common node become connected to each other.

Additionally, during graph creation it is possible to "fix" ("tie") to the
input vector layer any number of additional points. For each additional
point a match will be found --- closest graph vertex or closest graph edge.
In the latter case the edge will be splitted and new vertex added.

As the properties of the edge a vector layer attributes can be used and
length of the edge.

Converter from vector layer to graph developed using `Builder <http://en.wikipedia.org/wiki/Builder_pattern>`_
programming pattern. For graph construction response so-called Director.
There is only one Director for now: `QgsLineVectorLayerDirector <http://doc.qgis.org/api/classQgsLineVectorLayerDirector.html>`_.
The director sets the basic settings that will be used to construct a graph
from a line vector layer, used the builder to create graph. Currently, as
in the case with the director, only one builder exists: `QgsGraphBuilder <http://doc.qgis.org/api/classQgsGraphBuilder.html>`_,
that creates `QgsGraph <http://doc.qgis.org/api/classQgsGraph.html>`_ objects.
You may want to implement own builders that will build a graphs compatible
with such libraries as `BGL <http://www.boost.org/doc/libs/1_48_0/libs/graph/doc/index.html>`_
or `NetworkX <http://networkx.lanl.gov/>`_.

To calculate edge properties programming pattern `strategy <http://en.wikipedia.org/wiki/Strategy_pattern>`_
used. For now only `QgsDistanceArcProperter <http://doc.qgis.org/api/classQgsDistanceArcProperter.html>`_
strategy available, that takes into account the length of the route. You
can implement your own strategy that will use all necessary parameters.
For example, RoadGraph plugin uses strategy that compute traveling time
using edge length and speed value from attributes.

It's time to dive in process.

First of all to use this library we should import networkanalysis module::

from qgis.networkanalysis import *

Than create director::

# don't use information about roads direction from layer attributes,
# all roads are treated as two-ways roads
director = QgsLineVectorLayerDirector( vLayer, -1, '', '', '', 3 )

# use fied with index 5 as source of information about roads direction.
# unilateral roads with direct direction have attribute value "yes",
# unilateral roads with reverse direction — "1", and accordingly bilateral
# roads — "no". By defaults roads are treated as two-ways roads. This
# sheme can be used with OpenStreetMap data
director = QgsLineVectorLayerDirector( vLayer, 5, 'yes', '1', 'no', 3 )

In director's constructor we should pass vector layer, that will be used
as source for graph and information about allowed movement on each road
segment (unilateral or bilateral movement, direct or reverse direction).
Here is full list of this parameters:

 * vl — vector layer used to build graph
 * directionFieldId — index of the attribute table field, where information
   about roads directions stored. If -1, then don't use this info at all
 * directDirectionValue — field value for roads with direct direction
   (moving from first line point to last one)
 * reverseDirectionValue — field value for roads with reverse direction
   (moving from last line point to first one)
 * bothDirectionValue — field value for bilateral roads
   (for such roads we can move from first point to last and from last to first)
 * defaultDirection — default road direction. This value will be used for
   those roads where field directionFieldId is not set or have some value
   different from aboves.

Then it is necessary to create strategy for calculating edge properties::

  properter = QgsDistanceArcProperter()

And tell the director about this strategy::

  director.addProperter( properter )

Now we can create builder, which will create graph. QgsGraphBuilder constructor
takes several arguments:

* crs — coordinate reference system to use. Mandatory argument.
* otfEnabled — use  «on the fly» reprojection or no. By default true (use OTF).
* topologyTolerance — topological tolerance. Default value is 0.
* ellipsoidID — ellipsoid to use. By default "WGS84".

::

  # only CRS is set, all other values are defaults
  builder = QgsGraphBuilder( myCRS )

Also  we can set several points, which will be used in analysis. For example::

  startPoint = QgsPoint( 82.7112, 55.1672 )
  endPoint = QgsPoint( 83.1879, 54.7079 )

Now all in place so we can build graph and "tie" points to it::

  tiedPoints = director.makeGraph( builder, [ startPoint, endPoint ] )

Building graph can take some time (depends on number of features in layer and
layer size). tiedPoints is a list with coordinates of "tied" points. When
build operation is finished we can get graph and use it in analysis::

  graph = builder.graph()

With next code we can get indexes of our points::

  startId = graph.findVertex( tiedPoints[ 0 ] )
  endId = graph.findVertex( tiedPoints[ 1 ] )


Graph analysis
--------------

Networks analysis used to find answers on two questions: which vertices
are connected and how to find shortest path. To solve this problems network
analysis library provides Dijkstra's algorithm.

Dijkstra's algorithm finds the best route from one of the vertices of the
graph to all the others and the values of the optimization parameters.
The results can be represented as shortest path tree.

The shortest path tree is as oriented weighted graph (or more precise --- tree)
with next properties:

  * only one vertex have no incoming edges — the root of the tree
  * all other vertices have only one incoming edge
  * if vertex B is reachable from vertex A, then path from A to B is single
    available path and it is optimal (shortes) on this graph

To get shortest path tree use methods Use methods :func:`shortestTree` and
:func:`dijkstra` of `QgsGraphAnalyzer <http://doc.qgis.org/api/classQgsGraphAnalyzer.html>`_
class. It is recommended to use method :func:`dijkstra` because it works
faster and more efficiently uses memory.

The :func:`shortestTree` method is useful when you want to walk around
shortest path tree. It always create new graph object (QgsGraph) and accepts
three variables:

  * source — input graph
  * startVertexIdx — index of the point on tree (the root of the tree)
  * criterionNum — number of edge property to use (started from 0).

::

  tree = QgsGraphAnalyzer.shortestTree( graph, startId, 0 )

The :func:`dijkstra` method has same arguments, but returns two arrays.
In first array element i contains index of the incoming edge or -1 if there
are no incoming edges. In seconf array element i contains distance from
the root of the tree to vertex i or DOUBLE_MAX if vertex i is unreachable
from the root.

::

  (tree, cost) = QgsGraphAnalyzer.dijkstra( graph, startId, 0 )

Here is very simple code to display shortest path tree using graph created
with :func:`shortestTree` method (select linestring layer in TOC and replace
coordinates with yours one). **Warning**: use this code only as example,
it creates a lots of `QgsRubberBand <http://doc.qgis.org/api/classQgsRubberBand.html>`_
objects and may be slow on large datasets.

::

  from PyQt4.QtCore import *
  from PyQt4.QtGui import *

  from qgis.core import *
  from qgis.gui import *
  from qgis.networkanalysis import *

  vl = qgis.utils.iface.mapCanvas().currentLayer()
  director = QgsLineVectorLayerDirector( vl, -1, '', '', '', 3 )
  properter = QgsDistanceArcProperter()
  director.addProperter( properter )
  crs = qgis.utils.iface.mapCanvas().mapRenderer().destinationCrs()
  builder = QgsGraphBuilder( crs )

  pStart = QgsPoint( -0.743804, 0.22954 )
  tiedPoint = director.makeGraph( builder, [ pStart ] )
  pStart = tiedPoint[ 0 ]

  graph = builder.graph()

  idStart = graph.findVertex( pStart )

  tree = QgsGraphAnalyzer.shortestTree( graph, idStart, 0 )

  i = 0;
  while ( i < tree.arcCount() ):
    rb = QgsRubberBand( qgis.utils.iface.mapCanvas() )
    rb.setColor ( Qt.red )
    rb.addPoint ( tree.vertex( tree.arc( i ).inVertex() ).point() )
    rb.addPoint ( tree.vertex( tree.arc( i ).outVertex() ).point() )
    i = i + 1

Same thing but using :func:`dijkstra` method::

  from PyQt4.QtCore import *
  from PyQt4.QtGui import *

  from qgis.core import *
  from qgis.gui import *
  from qgis.networkanalysis import *

  vl = qgis.utils.iface.mapCanvas().currentLayer()
  director = QgsLineVectorLayerDirector( vl, -1, '', '', '', 3 )
  properter = QgsDistanceArcProperter()
  director.addProperter( properter )
  crs = qgis.utils.iface.mapCanvas().mapRenderer().destinationCrs()
  builder = QgsGraphBuilder( crs )

  pStart = QgsPoint( -1.37144, 0.543836 )
  tiedPoint = director.makeGraph( builder, [ pStart ] )
  pStart = tiedPoint[ 0 ]

  graph = builder.graph()

  idStart = graph.findVertex( pStart )

  ( tree, costs ) = QgsGraphAnalyzer.dijkstra( graph, idStart, 0 )

  for edgeId in tree:
    if edgeId == -1:
      continue
    rb = QgsRubberBand( qgis.utils.iface.mapCanvas() )
    rb.setColor ( Qt.red )
    rb.addPoint ( graph.vertex( graph.arc( edgeId ).inVertex() ).point() )
    rb.addPoint ( graph.vertex( graph.arc( edgeId ).outVertex() ).point() )

Finding shortest path
^^^^^^^^^^^^^^^^^^^^^

Для получения оптимального маршрута между двумя произвольными точками
используется следующий подход. Обе точки (начальная A и конечная B)
«привязываются» к графу на этапе построения, затем при помощи метода
shortestTree или dijkstra находится дерево кратчайших маршрутов с корнем
в начальной точке A. В этом же дереве находим конечную точку B и начинаем
спуск по дереву от точки B к точке А. В общем виде алгоритм можно записать
так:

    присвоим Т = B
    пока Т != A цикл
        добавляем в маршрут точку Т
        берем ребро, которое входит в точку Т
        находим точку ТТ, из которой это ребро выходит
        присваиваем Т = ТТ
    добавляем в маршрут точку А

На этом построение маршрута закончено. Мы получили инвертированный список
вершин (т.е. вершины идут в обратном порядке, от конечной точки к начальной),
которые будут посещены при движении по кратчайшему маршруту.

Посмотрите еще раз на дерево кратчайших путей и представьте, что вы можете
двигаться только против направления стрелочек. При движении из точки №7
мы рано или поздно попадем в точку №1 (корень дерева) и не сможем двигаться
дальше.

Вот работающий пример поиска кратчайшего маршрута для Консоли Python QGIS
(только замените координаты начальной и конечной точки на свои, а также
выделите слой дорог в списке слоёв карты) с использованием метода shortestTree

::

  from PyQt4.QtCore import *
  from PyQt4.QtGui import *

  from qgis.core import *
  from qgis.gui import *
  from qgis.networkanalysis import *

  vl = qgis.utils.iface.mapCanvas().currentLayer()
  director = QgsLineVectorLayerDirector( vl, -1, '', '', '', 3 )
  properter = QgsDistanceArcProperter()
  director.addProperter( properter )
  crs = qgis.utils.iface.mapCanvas().mapRenderer().destinationCrs()
  builder = QgsGraphBuilder( crs )

  pStart = QgsPoint( -0.835953, 0.15679 )
  pStop = QgsPoint( -1.1027, 0.699986 )

  tiedPoints = director.makeGraph( builder, [ pStart, pStop ] )
  graph = builder.graph()

  tStart = tiedPoints[ 0 ]
  tStop = tiedPoints[ 1 ]

  idStart = graph.findVertex( tStart )
  tree = QgsGraphAnalyzer.shortestTree( graph, idStart, 0 )

  idStart = tree.findVertex( tStart )
  idStop = tree.findVertex( tStop )

  if idStop == -1:
    print "Path not found"
  else:
    p = []
    while ( idStart != idStop ):
      l = tree.vertex( idStop ).inArc()
      if len( l ) == 0:
        break
      e = tree.arc( l[ 0 ] )
      p.insert( 0, tree.vertex( e.inVertex() ).point() )
      idStop = e.outVertex()

    p.insert( 0, tStart )
    rb = QgsRubberBand( qgis.utils.iface.mapCanvas() )
    rb.setColor( Qt.red )

    for pnt in p:
      rb.addPoint(pnt)

А вот пример с использованием метода dikstra

::

  from PyQt4.QtCore import *
  from PyQt4.QtGui import *

  from qgis.core import *
  from qgis.gui import *
  from qgis.networkanalysis import *

  vl = qgis.utils.iface.mapCanvas().currentLayer()
  director = QgsLineVectorLayerDirector( vl, -1, '', '', '', 3 )
  properter = QgsDistanceArcProperter()
  director.addProperter( properter )
  crs = qgis.utils.iface.mapCanvas().mapRenderer().destinationCrs()
  builder = QgsGraphBuilder( crs )

  pStart = QgsPoint( -0.835953, 0.15679 )
  pStop = QgsPoint( -1.1027, 0.699986 )

  tiedPoints = director.makeGraph( builder, [ pStart, pStop ] )
  graph = builder.graph()

  tStart = tiedPoints[ 0 ]
  tStop = tiedPoints[ 1 ]

  idStart = graph.findVertex( tStart )
  idStop = graph.findVertex( tStop )

  ( tree, cost ) = QgsGraphAnalyzer.dijkstra( graph, idStart, 0 )

  if tree[ idStop ] == -1:
    print "Path not found"
  else:
    p = []
    curPos = idStop
    while curPos != idStart:
      p.append( graph.vertex( graph.arc( tree[ curPos ] ).inVertex() ).point() )
      curPos = graph.arc( tree[ curPos ] ).outVertex();

    p.append( tStart )

    rb = QgsRubberBand( qgis.utils.iface.mapCanvas() )
    rb.setColor( Qt.red )

    for pnt in p:
      rb.addPoint(pnt)

Areas of the availability
^^^^^^^^^^^^^^^^^^^^^^^^^

Назовем областью доступности вершины графа А такое подмножество вершин
графа, доступных из вершины А, что стоимость оптимального пути от А до
элементов этого множества не превосходит некоторого заданного значения.

Более наглядно это определение можно объяснить на следующем примере:
«Есть пожарное депо. В какую часть города сможет попасть пожарная машина
в за 5 минут, 10 минут, 15 минут?». Ответом на этот вопрос и являются
области доступности пожарного депо.

Поиск областей доступности легко реализовать при помощи метода dijksta
класса QgsGraphAnalyzer. Достаточно сравнить элементы возвращаемого значения
с заданным параметром. Если величина cost[ i ] меньше заданного параметра
или равна ему, тогда i-я вершина графа принадлежит множеству доступности,
в противном случае — не принадлежит.

Не столь очевидным является нахождение границ доступности. Нижняя граница
доступности — множество вершин которые еще можно достигнуть, а верхняя
граница — множество вершин которых уже нельзя достигнуть. На самом деле
все просто: граница доступности проходит по таким ребрам дерева кратчайших
путей, для которых вершина-источник ребра доступна, а вершина-цель недоступна.

Here is example::

  from PyQt4.QtCore import *
  from PyQt4.QtGui import *

  from qgis.core import *
  from qgis.gui import *
  from qgis.networkanalysis import *

  vl = qgis.utils.iface.mapCanvas().currentLayer()
  director = QgsLineVectorLayerDirector( vl, -1, '', '', '', 3 )
  properter = QgsDistanceArcProperter()
  director.addProperter( properter )
  crs = qgis.utils.iface.mapCanvas().mapRenderer().destinationCrs()
  builder = QgsGraphBuilder( crs )

  pStart = QgsPoint( 65.5462, 57.1509 )
  delta = qgis.utils.iface.mapCanvas().getCoordinateTransform().mapUnitsPerPixel() * 1

  rb = QgsRubberBand( qgis.utils.iface.mapCanvas(), True )
  rb.setColor( Qt.green )
  rb.addPoint( QgsPoint( pStart.x() - delta, pStart.y() - delta ) )
  rb.addPoint( QgsPoint( pStart.x() + delta, pStart.y() - delta ) )
  rb.addPoint( QgsPoint( pStart.x() + delta, pStart.y() + delta ) )
  rb.addPoint( QgsPoint( pStart.x() - delta, pStart.y() + delta ) )

  tiedPoints = director.makeGraph( builder, [ pStart ] )
  graph = builder.graph()
  tStart = tiedPoints[ 0 ]

  idStart = graph.findVertex( tStart )

  ( tree, cost ) = QgsGraphAnalyzer.dijkstra( graph, idStart, 0 )

  upperBound = []
  r = 2000.0
  i = 0
  while i < len(cost):
    if cost[ i ] > r and tree[ i ] != -1:
      outVertexId = graph.arc( tree [ i ] ).outVertex()
      if cost[ outVertexId ] < r:
        upperBound.append( i )
    i = i + 1

  for i in upperBound:
    centerPoint = graph.vertex( i ).point()
    rb = QgsRubberBand( qgis.utils.iface.mapCanvas(), True )
    rb.setColor( Qt.red )
    rb.addPoint( QgsPoint( centerPoint.x() - delta, centerPoint.y() - delta ) )
    rb.addPoint( QgsPoint( centerPoint.x() + delta, centerPoint.y() - delta ) )
    rb.addPoint( QgsPoint( centerPoint.x() + delta, centerPoint.y() + delta ) )
    rb.addPoint( QgsPoint( centerPoint.x() - delta, centerPoint.y() + delta ) )
