# tensorflow-ue4

[![GitHub release](https://img.shields.io/github/release/getnamo/tensorflow-ue4/all.svg)](https://github.com/getnamo/tensorflow-ue4/releases)
[![Github All Releases](https://img.shields.io/github/downloads/getnamo/tensorflow-ue4/total.svg)](https://github.com/getnamo/tensorflow-ue4/releases)

Unreal Engine plugin for [TensorFlow](https://www.tensorflow.org/). Enables training and implementating state of the art machine learning algorithms for your unreal projects. 

This plugin source contains C++, Blueprint and python scripts that encapsulate TensorFlow operations as an _Actor Component_. The plugin depends on [UnrealEnginePython](https://github.com/getnamo/UnrealEnginePython) plugin fork and [SocketIO Client](https://github.com/getnamo/socketio-client-ue4) plugin.

Releases for this plugin contain compiled versions of all dependency plugins and you should be able to drag and drop it into your project.

Currently early API, if you got ideas and fixes, consider contributing! See https://github.com/getnamo/tensorflow-ue4/issues for details. 

## Installation & Setup

 1.	[Install TensorFlow pre-requisites](https://www.tensorflow.org/install/install_windows) if you're using GPUs (NVIDIA cuDNN)
 2.	[Download Latest Release](https://github.com/getnamo/tensorflow-ue4/releases)
 3.	Create new or choose project.
 4.	Browse to your project folder (typically found at Documents/Unreal Project/{Your Project Root})
 5.	Copy *Plugins* folder into your Project root.
 6.	Enable the plugin via Edit->Plugins. Scroll down to Project->Networking. Click Enabled.
 7.	Restart the Editor and open your project again. Plugin is now ready to use.

## Note on Dependencies
Depends on [UnrealEnginePython](https://github.com/getnamo/UnrealEnginePython) plugin fork and [SocketIO Client](https://github.com/getnamo/socketio-client-ue4) plugin.

The UnrealEnginePython plugin fork contains changes to enable multi-threading, python script plugin encapsulation and automatic dependency resolution via pip. Simply specifying tensorflow as _pythonModule_ dependency in https://github.com/getnamo/tensorflow-ue4/blob/master/Content/Scripts/upymodule.json makes the editor auto-resolve the dependency on first run. The multi-threading support contains a callback system allowing long duration operations to happen on a background thread (e.g. training) and then receiving callbacks on your game-thread. This enables TensorFlow to work without noticeably impacting the game thread.

SocketIO Client is used for easy conversion between C++ and blueprint types (structs and variables) and JSON. Can also be used optionally to easily connect to a real-time web service via [socket.io](https://socket.io/).

## Examples
Project examples found under [tensorflow-ue4-examples](https://github.com/getnamo/tensorflow-ue4-examples). 

It is also the main repository where all development is tracked for all plugin dependencies for tensorflow.

## Python API

You can either train directly or use a trained model inside UE4.

To start, add your python script file to _{Project Root Folder}/Content/Scripts_.

wrap your tensorflow python code by subclassing TFPluginAPI.

#### MySubClass(TFPluginAPI)

import the following and subclass in your module file

```python
import tensorflow as tf
import unreal_engine as ue
from TFPluginAPI import TFPluginAPI
```

Add the following class functions

```python
class ExampleAPI(TFPluginAPI):

	#expected optional api: setup your model for training
	def setup(self):
		pass
		
	#expected optional api: parse input object and return a result object, which will be converted to json for UE4
	def runJsonInput(self, jsonInput):
		result = {}
		return result

	#expected optional api: start training your network
	def train(self):
		pass
    
#NOTE: this is a module function, not a class function. Change your CLASSNAME to reflect your class
#required function to get our api
def getApi():
	#return CLASSNAME.getInstance()
	return ExampleAPI.getInstance()
```

Note the ```getApi()``` module function which needs to return a matching instance of your defined class. The rest of the functionality depends on what API you wish to use for your use case. At the moment the plugin supports input/output from UE4 via JSON encoding.

If you have a trained model, simply setup your model/load it from disk and omit the training function, and forward your evaluation/input via ```runJsonInput(jsonArgs)```

If you wish to train in UE4, ensure you check for ```self.shouldstop``` after each batch/epoch to handle early exit requests from the user e.g. when you _EndPlay_.

A slightly more expanded example api:

```python
class ExampleAPI(TFPluginAPI):

	#expected optional api: setup your model for training
	def setup(self):
		#setup or load your model and pass it into stored
		
		#Usually store session, graph, and model if using keras
		self.sess = tf.InteractiveSession()
		self.graph = tf.get_default_graph()

	#expected optional api: parse input object and return a result object, which will be converted to json for UE4
	def runJsonInput(self, jsonInput):
		#e.g. our json input could contain a pixel array which you could set your embedd into a feed_dict
		#pixelarray = jsonInput['pixels']
		#feed_dict = {self.model['x']: [pixelarray]}

		#run input on your graph, you may need to use numpy to reshape the input to fit your model format
		#e.g. sess.run(model['y'], feed_dict)
		# where y is your result graph and feed_dict is {x:[input]}

		#...

		#return a json you will parse in blueprint e.g. a prediction
		result = {}
		result['prediction'] = -1

		return result

	#expected optional api: no params forwarded for training? TBC
	def train(self):
		#train here

		#...

		#inside your training loop check if we should stop early typically per batch
		#if(this.shouldstop):
		#	break
		pass

#required function to get our api
def getApi():
	#return CLASSNAME.getInstance()
	return MnistSimple.getInstance()

```

a full example can be seen here: https://github.com/getnamo/tensorflow-ue4-examples/blob/master/Content/Scripts/mnistSimple.py

## Blueprint API

### Load your python module from your TensorflowComponent
Once you've [written your python module](https://github.com/getnamo/tensorflow-ue4#mysubclasstfpluginapi), Select your TensorflowComponent inside your actor blueprint

![select component](http://i.imgur.com/f9Syql1.png)

and change the TensorFlowModule name to reflect your _filename_ without .py. e.g. if my python file is _ExampleAPI.py_ it would look like this

![change module name](http://i.imgur.com/mpzymgd.png)

Optionally disable the verbose python log and change other toggles such as training on _BeginPlay_ or disabling multithreading (not recommended).

### Training

By default the _train()_ function get's called on the component's begin play call. You can optionally untick this option and call _Begin Training_ manually.

![manual train](http://i.imgur.com/YM3KZwy.png)

### Sending Tensor inputs to your model

You control what type of data you forward to your python module and the only limitation for v0.1 api is that it should be JSON formatted.

#### Basic Json String
In the simplest case you can send e.g. a basic json string ```{"StringData","some string"}``` constructed using SIOJson like so

![send json string](http://i.imgur.com/rVYw7we.png)

#### Any UStruct Example

The most common way of sending input is to interweave a struct in a json object. In this particular example we send a vector, but you can easily make a custom struct type (can be completely defined in blueprint!), fill its data and replace the vector in the graph for the desired result.

![send struct](http://i.imgur.com/3WLXgqL.png)

#### Special convenience case: UTexture2D

A convenience function wraps a UTexture2D into a json object with ```{"pixels":[<1D array of pixels>]}``` which you can reshape using numpy.

![send texture](http://i.imgur.com/WNLG3Z1.png)

### Handling Tensorflow Events

Select your _Tensorflow Component_ from your actor blueprint and then click + to subscribe to the chosen event in the event graph. 

![events](http://i.imgur.com/FsSUJTj.png)

v0.1 api supports the following events

#### On Results

Called when _runJsonInput()_ completes on your python module. The returned data is a json string of the return data you pass at the end of the function

![onresults](http://i.imgur.com/VYGjTUw.png)

Typically you'd want to convert this string into _SIOJsonObject_ so you can use your results data in blueprint.

#### On Training Complete

When you _Train()_ call is complete you receive this event with ```{'elapsed':<time taken>}``` json, optionally with additional data passed as return data from your function.

![ontraining](http://i.imgur.com/XiZhH04.png)


## [License](https://github.com/getnamo/tensorflow-ue4/blob/master/LICENSE)
Plugin - [MIT](https://opensource.org/licenses/MIT)

TensorFlow and TensorFlow Icon - [Apache 2.0](http://www.apache.org/licenses/LICENSE-2.0)



