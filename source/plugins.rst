
.. _plugins:

Developing Python Plugins
=========================


It is possible to create plugins in Python programming language. In comparison with classical plugins written in C++ these should be easier to write,
understand, maintain and distribute due the dynamic nature of the Python language.

Python plugins are listed together with C++ plugins in QGIS plugin manager. They're being searched for in these paths:

    * UNIX/Mac: :file:`~/.qgis/python/plugins` and :file:`(qgis_prefix)/share/qgis/python/plugins`
    * Windows: :file:`~/.qgis/python/plugins` and :file:`(qgis_prefix)/python/plugins`

Home directory (denoted by above :file:`~`) on Windows is usually something like :file:`C:\\Documents and Settings\\(user)`.
Subdirectories of these paths are considered as Python packages that can be imported to QGIS as plugins.

Steps:

1. *Idea*: Have an idea about what you want to do with your new QGIS plugin.
   Why do you do it?
   What problem do you want to solve?
   Is there already another plugin for that problem?

2. *Create files*: Create the files described next.
   A starting point (:file:`__init.py__`).
   A main python plugin body (:file:`plugin.py`).
   A form in QT-Designer (:file:`form.ui`), with its :file:`resources.qrc`.

3. *Write code*: Write the code inside the :file:`plugin.py`

4. *Test*: Close and re-open QGIS and import your plugin again. Check if everything is OK.

5. *Publish*: Publish your plugin in QGIS repository or make your own repository as an "arsenal" of personal "GIS weapons" 


Writing a plugin
----------------

Since the introduction of python plugins in QGIS, a number of plugins have appeared -
on `Plugin Repositories wiki page <http://www.qgis.org/wiki/Python_Plugin_Repositories>`_ you can find some of them,
you can use their source to learn more about programming with PyQGIS or find out whether you are not duplicating development effort.
Ready to create a plugin but no idea what to do? `Python Plugin Ideas wiki page <http://www.qgis.org/wiki/Python_Plugin_Ideas>`_ lists wishes from the community!


Creating necessary files
------------------------

Here's the directory structure of our example plugin::

  PYTHON_PLUGINS_PATH/
    testplug/
      __init__.py
      plugin.py
      resources.qrc
      resources.py
      form.ui
      form.py

What is the meaning of the files:

* __init__.py = The starting point of the plugin. Contains general info, version, name and main class.
* plugin.py = The main working code of the plugin. Contains all the information about the actions of the plugin and the main code.
* resources.qrc = The .xml document created by QT-Designer. Contains relative paths to resources of the forms.
* resources.py = The translation of the .qrc file described above to Python.
* form.ui = The GUI created by QT-Designer.
* form.py = The translation of the form.ui described above to Python. 

`Here <http://pyqgis.org/builder/plugin_builder.py>`_ and `there <http://www.dimitrisk.gr/qgis/creator/>`_ are two automated ways of creating
the basic files (skeleton) of a typical QGIS Python plugin. Useful to help you start with a typical plugin.

Writing code
------------

__init__.py
^^^^^^^^^^^

First, plugin manager needs to retrieve some basic information about the plugin such as its name, description etc.
File :file:`__init__.py` is the right place where to put this information::

  def name():
    return "My testing plugin"
  
  def description():
    return "This plugin has no real use."
  
  def version():
    return "Version 0.1"
  
  def qgisMinimumVersion(): 
    return "1.0"
  
  def authorName():
    return "Developer"
  
  def classFactory(iface):
    # load TestPlugin class from file testplugin.py
    from testplugin import TestPlugin
    return TestPlugin(iface)

plugin.py
^^^^^^^^^

One thing worth mentioning is ``classFactory()`` function which is called when the plugin gets loaded to QGIS.
It receives reference to instance of :class:`QgisInterface` and must return instance of your plugin - in our case it's called ``TestPlugin``.
This is how should this class look like (e.g. :file:`testplugin.py`)::

  from PyQt4.QtCore import *
  from PyQt4.QtGui import *
  from qgis.core import *

  # initialize Qt resources from file resouces.py
  import resources

  class TestPlugin:

    def __init__(self, iface):
      # save reference to the QGIS interface
      self.iface = iface

    def initGui(self):
      # create action that will start plugin configuration
      self.action = QAction(QIcon(":/plugins/testplug/icon.png"), "Test plugin", self.iface.mainWindow())
      self.action.setWhatsThis("Configuration for test plugin")
      self.action.setStatusTip("This is status tip")
      QObject.connect(self.action, SIGNAL("triggered()"), self.run)

      # add toolbar button and menu item
      self.iface.addToolBarIcon(self.action)
      self.iface.addPluginToMenu("&Test plugins", self.action)

      # connect to signal renderComplete which is emitted when canvas rendering is done
      QObject.connect(self.iface.mapCanvas(), SIGNAL("renderComplete(QPainter *)"), self.renderTest)

    def unload(self):
      # remove the plugin menu item and icon
      self.iface.removePluginMenu("&Test plugins",self.action)
      self.iface.removeToolBarIcon(self.action)

      # disconnect form signal of the canvas
      QObject.disconnect(self.iface.MapCanvas(), SIGNAL("renderComplete(QPainter *)"), self.renderTest)

    def run(self):
      # create and show a configuration dialog or something similar
      print "TestPlugin: run called!"

    def renderTest(self, painter):
      # use painter for drawing to map canvas
      print "TestPlugin: renderTest called!"


Only functions of the plugin that must exist are ``initGui()`` and ``unload()``.
These functions are called when plugin is loaded and unloaded.

Resource File
^^^^^^^^^^^^^

You can see that in ``initGui()`` we've used an icon from the resource file (called :file:`resources.qrc` in our case)::

  <RCC>
    <qresource prefix="/plugins/testplug" >
       <file>icon.png</file>
    </qresource>
  </RCC>

It is good to use a prefix that will not collide with other plugins or any parts of QGIS, otherwise you might get resources you did not want.
Now you just need to generate a Python file that will contain the resources. It's done with :command:`pyrcc4` command::

  pyrcc4 -o resources.py resources.qrc

And that's all... nothing complicated :)
If you've done everything correctly you should be able to find and load your plugin in plugin manager and see a message in console
when toolbar icon or appopriate menu item is selected.

When working on a real plugin it's wise to write the plugin in another (working) directory and create a makefile which will
generate UI + resource files and install the plugin to your QGIS installation.

Documentation
-------------

*This documentation method requires Qgis version 1.5.*

The documentation for the plugin can be written as HTML help files. The :mod:`qgis.utils` module provides a function, :func:`showPluginHelp`
which will open the help file users browser, in the same way as other QGIS help.

The :func:`showPluginHelp`` function looks for help files in the same directory as the calling module.
It will look for, in turn, :file:`index-ll_cc.html`, :file:`index-ll.html`, :file:`index-en.html`, :file:`index-en_us.html` and :file:`index.html`,
displaying whichever it finds first. Here ``ll_cc`` is the QGIS locale. This allows multiple translations of the documentation to be included with the plugin.

The :func:`showPluginHelp` function can also take parameters packageName, which identifies a specific plugin for which the help will be displayed,
filename, which can replace "index" in the names of files being searched, and section, which is the name of an html anchor tag in the document
on which the browser will be positioned.

Code Snippets
-------------

This section features code snippets to facilitate plugin development.

How to call a method by a key shortcut
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

In the plug-in add to the ``initGui()``::

  self.keyAction = QAction("Test Plugin", self.iface.mainWindow())
  self.iface.registerMainWindowAction(self.keyAction, "F7") # action1 is triggered by the F7 key
  self.iface.addPluginToMenu("&Test plugins", self.keyAction)
  QObject.connect(self.keyAction, SIGNAL("triggered()"),self.keyActionF7)

To ``unload()`` add::

  self.iface.unregisterMainWindowAction(self.keyAction)

The method that is called when F7 is pressed::

  def keyActionF7(self):
    QMessageBox.information(self.iface.mainWindow(),"Ok", "You pressed F7")

How to toggle Layers (work around)
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

*Note:* from QGIS 1.5 there is :class:`QgsLegendInterface` class that allows some manipulation with list of layers within legend.

As there is currently no method to directly access the layers in the legend, here is a workaround how to toggle the layers using layer transparency::

  def toggleLayer(self, lyrNr):
    lyr = self.iface.mapCanvas().layer(lyrNr)
    if lyr:
      cTran = lyr.getTransparency()
      lyr.setTransparency(0 if cTran > 100 else 255)
      self.iface.mapCanvas().refresh()	

The method requires the layer number (0 being the top most) and can be called by::

  self.toggleLayer(3)

How to access attribute table of selected features
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

::

  def changeValue(self, value):
    layer = self.iface.activeLayer()
    if(layer):		
      nF = layer.selectedFeatureCount()
      if (nF > 0):		
      layer.startEditing()
      ob = layer.selectedFeaturesIds()
      b = QVariant(value)
      if (nF > 1):
        for i in ob:
        layer.changeAttributeValue(int(i),1,b) # 1 being the second column
      else:
        layer.changeAttributeValue(int(ob[0]),1,b) # 1 being the second column
      layer.commitChanges()
      else:
        QMessageBox.critical(self.iface.mainWindow(),"Error", "Please select at least one feature from current layer")
    else:
      QMessageBox.critical(self.iface.mainWindow(),"Error","Please select a layer")
  

The method requires the one parameter (the new value for the attribute field of the selected feature(s)) and can be called by::

  self.changeValue(50)


How to debug a plugin using PDB
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

First add this code in the spot where you would like to debug::

 # Use pdb for debugging
 import pdb
 # These lines allow you to set a breakpoint in the app
 pyqtRemoveInputHook()
 pdb.set_trace()

Then run QGIS from the command line.

On Linux do:

:command:`$ ./Qgis`

On Mac OS X do:

:command:`$ /Applications/Qgis.app/Contents/MacOS/Qgis`

And when the application hits your breakpoint you can type in the console!

Testing
-------

Releasing the plugin
--------------------

Once your plugin is ready and you think the plugin could be helpful for some people, do not hesitate to upload it to `PyQGIS plugin repository <http://pyqgis.org/>`_.
On that page you can find also packaging guidelines how to prepare the plugin to work well with the plugin installer.
Or in case you would like to set up your own plugin repository, create a simple XML file that will list the plugins and their metadata,
for examples see other `plugin repositories <http://www.qgis.org/wiki/Python_Plugin_Repositories>`_.

Remark: Configuring Your IDE on Windows
---------------------------------------

On Linux there is no additional configuration needed to develop plug-ins. But on Windows you need to make sure you that you have the same
environment settings and use the same libraries and interpreter as QGIS. The fastest way to do this, is to modify the startup batch file of QGIS.

If you used the OSGeo4W Installer, you can find this under the bin folder of your OSGoeW install. Look for something like :file:`C:\\OSGeo4W\\bin\\qgis-unstable.bat`.

I will illustrate how to set up the `Pyscripter IDE <http://code.google.com/p/pyscripter>`_. Other IDE’s might require a slightly different approach:

* Make a copy of qgis-unstable.bat and rename it pyscripter.bat.
* Open it in an editor. And remove the last line, the one that starts qgis.
* Add a line that points to the your pyscripter executable and add the commandline argument that sets the version of python to be used, in version 1.3 of qgis this is python 2.5.
* Also add the argument that points to the folder where pyscripter can find the python dll used by qgis, you can find this under the bin folder of your OSGeoW install::

    @echo off
    SET OSGEO4W_ROOT=C:\OSGeo4W
    call "%OSGEO4W_ROOT%"\bin\o4w_env.bat
    call "%OSGEO4W_ROOT%"\bin\gdal16.bat
    @echo off
    path %PATH%;%GISBASE%\bin
    Start C:\pyscripter\pyscripter.exe --python25 --pythondllpath=C:\OSGeo4W\bin

Now when you double click this batch file and it will start pyscripter. 
