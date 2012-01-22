.. index:: отрисовка карты, печать карты

.. _composer:

Отрисовка карты и печать
========================

Существует два способа получить печатную карту из исходных данных: простой
и быстрый используя :class:`QgsMapRenderer` или создание и тщательная
настройка компоновки используя :class:`QgsComposition` и сопутствующие классы.

.. index:: отрисовка карты; простая

Простая отрисовка
-----------------

Отрисовка нескольких слоёв с помощью :class:`QgsMapRenderer` --- создаётся
объект для рисования (``QImage``, ``QPainter`` и др.), задаётся набор слоёв,
охват, размер результата и выполняется рендеринг::

  # создаём изображение
  img = QImage(QSize(800,600), QImage.Format_ARGB32_Premultiplied)

  # устанавливаем цвет фона
  color = QColor(255,255,255)
  img.fill(color.rgb())

  # create painter
  p = QPainter()
  p.begin(img)
  p.setRenderHint(QPainter.Antialiasing)

  render = QgsMapRender()

  # задаём набор слоёв
  lst = [ layer.getLayerID() ]  # добавляем ID необходимых слоёв
  render.setLayerSet(lst)

  # устанавливаем охват
  rect = QgsRect(render.fullExtent())
  rect.scale(1.1)
  render.setExtent(rect)

  # устанавливаем размер результата
  render.setOutputSize(img.size(), img.logicalDpiX())

  # выполняем отрисовку
  render.render(p)

  p.end()

  # сохраняем изображение
  img.save("render.png","png")

.. index:: вывод; использование компоновщика карт

Вывод с использованием компоновщика карт
----------------------------------------

Компоновщик карт это удобный инструмент для создания более сложных печатных
карт, по сравнению с простой отрисовкой описанной выше. Используя компоновщик
можно создавать составные компоновки, содержащие карты, подписи, легенду,
таблицы и другие элементы, которые обычно присутсвуют на печатных картах.
Готовую компоновку можно экспортировать в PDF, растровое изображение или
сразу же распечатать на принтере.

Компоновщик состоит из множества классов. Все они являются частью библиотеки
ядра. В QGIS присутствует удобный интерфейс пользователя для расстановки
всех элементов, но он пока ещё не доступен через библиотеку графического
интерфейса. Если вы не знакомы с `Qt Graphics View framework <http://doc.qt.nokia.com/stable/graphicsview.html>`_,
рекомендуем изучить документацию сейчас, потому что компоновщик основан на
этом фреймворке.

Основным классом компоновщика является :class:`QgsComposition`, который
в свою очередь основан на :class:`QGraphicsScene`. Создадим экземпляр этого
класса::

  mapRenderer = iface.mapCanvas().mapRenderer()
  c = QgsComposition(mapRenderer)
  c.setPlotStyle(QgsComposition.Print)

Обратите вниманием, что компоновка принимает в качестве параметра экземпляр
:class:`QgsMapRenderer`. Предполагается, что код выполняется в среде QGIS
и поэтому используется рендер активной карты. Компоновка использует различные
параметры рендера, и самое главное --- набор слоёв карты и текущий охват. При
использовании компоновщика в самостоятельном приложении необходимо создать
свой собственный экземпляр рендера, как это показано в разделе выше, и передать
его в компоновку.

К компоновке можно добавлять разные элементы (карту, подписи, ...) --- все
они являются потомками класса :class:`QgsComposerItem`. В настоящее время
доступны следующие элементы:

* карта --- этот элемент определяет положение карты. Так можно создать карту
  и растянуть её на весь лист::

    x, y = 0, 0
    w, h = c.paperWidth(), c.paperHeight()
    composerMap = QgsComposerMap(c, x, y, w, h)
    c.addItem(composerMap)

* подпись --- позволяет отображать подписи. Можно изменять шрифт, цвет,
  выравнивание и поля
  ::

    composerLabel = QgsComposerLabel(c)
    composerLabel.setText("Hello world")
    composerLabel.adjustSizeToText()
    c.addItem(composerLabel)

* легенда
  ::

    legend = QgsComposerLegend(c)
    legend.model().setLayerSet(mapRenderer.layerSet())
    c.addItem(legend)

* масштабная линейка
  ::

    item = QgsComposerScaleBar(c)
    item.setStyle('Numeric') # при желании стиль можно изменить
    item.setComposerMap(composerMap)
    item.applyDefaultSize()
    c.addItem(item)

* стрелка севера
* изображение
* фигура
* таблица

По умолчанию только что созданные элементы компоновки имеют нулевое положение
(левый верхний угол страницы) и нулевой размер. Положение и размер всегда
задаюся в миллиметрах::

  # расположить подпись на расстоянии 1 см от верхнего края и 2 см от левого края страницы
  composerLabel.setItemPosition(20,10)
  # установить размер и положение метки (ширина 10 см, высота 3 см)
  composerLabel.setItemPosition(20,10, 100, 30)

Вокруг каждого элемента по умолчанию рисуется рамка. Убрать её можно так::

  composerLabel.setFrame(False)

Помимо создания элементов компоновки вручную QGIS поддерживает шаблоны
компоновок, которые являются компоновками со всеми элементами, сохраненными
в файл .qpt (формат XML). К сожалению, этот функционал пока ещё не доступен
в API.

После того как компоновка готова (все элементы созданы и добавлены к компоновке),
можно приступать к представлению результатов в растровой или векторной форме.

По умолчанию для вывода используется лист формата A4 и разрешение 300 DPI.
При необходимости эти настройки можно изменить. Размер бумаги задаётся в миллиметрах::

  c.setPaperSize(width, height)
  c.setPrintResolution(dpi)

.. index:: вывод; растровое изображение

Вывод в растровое изображение
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Следующий фрагмент кода показывает как вывести компоновку в растровое изображение::

  dpi = c.printResolution()
  dpmm = dpi / 25.4
  width = int(dpmm * c.paperWidth())
  height = int(dpmm * c.paperHeight())

  # создаём выходное изображение и инициализируем его
  image = QImage(QSize(width, height), QImage.Format_ARGB32)
  image.setDotsPerMeterX(dpmm * 1000)
  image.setDotsPerMeterY(dpmm * 1000)
  image.fill(0)

  # отрисовываем компоновку
  imagePainter = QPainter(image)
  sourceArea = QRectF(0, 0, c.paperWidth(), c.paperHeight())
  targetArea = QRectF(0, 0, width, height)
  c.render(imagePainter, targetArea, sourceArea)
  imagePainter.end()

  image.save("out.png", "png")

.. index:: вывод; PDF

Вывод в формате PDF
~~~~~~~~~~~~~~~~~~~

Следующий пример иллюстрирует вывод компоновки в файл формата PDF::

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


