# Design document

MapTextSynthesizer is a program to dynamically generate synthetic images containing text, which appear to be from historical maps. The produced images serve as training data for a Convolutional Neural Network that recognizes text in scanned images of historical maps. (Data not intended for training a text spotting model)

## General Problem & Approach

When trained on a static training dataset of scene text ([MJSynth](http://www.robots.ox.ac.uk/~vgg/data/text/)), a Convolutional Neural Network built by Professor [Jerod Weinman](https://github.com/weinman/cnn_lstm_ctc_ocr) that recognizes text in historical maps has a 30% character error rate and a 50% word error rate. This is suboptimal, and we believe that accuracy can be improved by using training data that better fits the purpose of the CNN.

The current, static dataset, MJSynth is gigantic in size (10Gb), which makes distributing and using it cumbersome. By nature, it also has a limited amount of training data, which makes the text recognizer prone to overfitting to the data. Additionally, the data isn’t specifically intended for training a recognizer to read historical maps. The majority of images in the MJSynth dataset do not accurately reflect the population of text styles and background noise in maps.

To resolve these issues, we have created a program to dynamically generate synthetic, map-like images containing text  and use those images as training data for Professor Weinman’s map-text recognizing CNN. 
In contrast with the static dataset, our dynamic text image generator occupies only 6.3M disk memory while capable of producing a theoretically infinite amount of training data, all of which is specifically engineered to look similar to text found in historical maps.

## Files Developed

```
|
|-include/mtsynth/map_text_synthesizer.hpp (public header)
|
|-inc/                                     (private headers)
|       |-mts_implementation.hpp
|       |-mts_basehelper.hpp
|       |-mts_texthelper.hpp
|       |-mts_bghelper.hpp
|       |-mts_config.hpp
|
|-src/
|       |-map_text_synthesizer.cpp
|       |-mts_implementation.cpp
|       |-mts_basehelper.cpp
|       |-mts_texthelper.cpp
|       |-mts_bghelper.cpp
|       |-mts_config.cpp
```

### Why this architecture?

Our classes divide the components of the map text synthesizer as sensibly as possible. Text attributes and background features are the synthesizer’s main functionality, so their support methods are divided into the ```MTS_TextHelper``` class and ```MTS_BackgroundHelper``` classes respectively. Methods shared between text attributes and background features are placed in the ```MTS_BaseHelper``` class for ease of access; every other class includes a pointer to ```MTS_BaseHelper``` as a class member.

### What do these files do

##### map_text_synthesizer.hpp:
The public header of this software. Also the header file of ```MapTextSynthesizer``` class. Exposes public methods for users to create a synthesizer, set the candidate fonts, set the candidate captions, and get the generated label and image.

##### map_text_synthesizer.cpp:
The source file of ```MapTextSynthesizer``` class. The static create() method returns a pointer to an instance of ```MTSImplementation``` class.

##### mts_implementation.hpp/mts_implementation.cpp:
The header and source files of ```MTSImplementation``` class. This class is a subclass of ```MapTextSynthesizer``` class, and is used to hide implementation details of the synthesizer. This class calls upon ```MTS_*Helper``` classes to generate a cairo surface which contains a map text image. Then the cairo surface will be converted to an OpenCV mat object, go through some additional processing such as Gaussian noise and Gaussian blur, and finally be returned to the user. This class is also responsible for parsing the config file into a hashmap, constructing a ```MTS_BaseHelper``` instance with that hashmap, and pass pointer to the ```MTS_BaseHelper``` instance to ```MTS_TextHelper``` and ```MTS_BackgroundHelper``` class.

##### mts_basehelper.hpp/mts_basehelper.cpp:
The header and source files of the ```MTS_BaseHelper``` class. Being a shared location, it houses the hashmap of user configured parameter values, two random number generators and the shared methods among all the other classes.

##### mts_bghelper.hpp/mts_bghelper.cpp:
The header and source files of the ```MTS_BackgroundHelper``` class. They contain the definitions and implementation for all unshared background generating methods that do not need to be exposed to the user. Handles drawing of lines, textures, and the background bias field in cairo.

##### mts_texthelper.hpp/mts_texthelper.cpp:
The header and source files of the ```MTS_TextHelper``` class. They contain the definitions and implementation for all unshared text generating methods that do not need to be exposed to the user. Handles creation of the main text attributes and distracting text in pango and cairo.

##### mts_config.hpp/mts_config.cpp:  
The header and source files of the ```MTSConfig``` class. The class handles all fetching and storage of user configurable parameters from a text file. It also managest the distribution of those variablse to the classes that use the values. 


## How to Configure MapTextSynthesizer

Using the same tried-and-true parameters for the synthesizer as we have is guaranteed to get relatively realistic results. However, should you want to experiment, the tools are available.

#### Parameters
There are over 100 user configurable parameters that change the effects and probabilities of features, all changeable in a plain text file for readability and ease of use. The parameter text file can be found from the base directory at samples/config.txt. 
Simply change any desired values in the text file (values should all be doubles), save them, and then recompile and run a sample. Some changes may result in only miniscule differences.
#### Fonts
Available fonts can also be altered through the config file. Simply change the value of the "fonts" parameter, which takes in a comma-separated list of file names. Each font file should contain one font family per line.

If the google fonts repository we use isn’t desirable, fonts can be added or removed by simply adding or removing the target font name to/from one of the three font files: blocky.txt, regular.txt, and cursive.txt. The font population is divided between these files to control the representation of each category in our images.

When adding font names, be sure that you have the fonts installed on your machine; if the synthesizer encounters a font that isn’t on your machine while processing the available fonts, it will exit the process with an terminal message.
Fonts must be downloaded into a .fonts folder in home directory to be visible.

#### Words 
The population of words that are displayed in the synthesized images is also configurable through config file. The "captions" parameter also takes in a comma-separated list of filenames. Each caption file should contain one caption per line. We have included a directory containing files of Iowa place-names for generating realistic map samples, but this does not have to be the source. To set your own list of captions, do

If no captions are found/supplied, the synthesizer will generate its own captions of random characters and length. 


## Notes for Developers/ Contributors

### How to add a custom text attribute

Since our text shape is rendered using pangocairo, you will have to familiarize yourself with the pango and cairo libraries first in order to integrate your own text attributes.

The ```generateTextPatch()``` method in ```MTS_TextHelper``` is the main method for generating a text image to a cairo surface.

If your text attribute is controllable by ```PangoLayout```, then you can directly add it to the layout object in ```generateTextPatch()```. If your text attribute is about transforming the text shape through matrix transformation, you can apply cairo’s matrix transformation methods such as ```cairo_rotate()``` and ```cairo_scale()``` to the cairo surface.

### How to add a custom background feature

The enumerated type ```BGFeature``` in the ```MTS_BaseHelper``` class is used to hold all types of background features that could be added to an image.

The ```generateBgFeatures()``` method in ```MTS_BackgroundHelper``` is used to sample a subset of background features from those available in ```BGFeature```.

The ```generateBgSample()``` method in ```MTS_BackgroundHelper``` is the main method for generating the background cairo surface given a vector of ```BGFeatures``` to generate.

### How to integrate the synthesizer with Tensorflow

The files found in the `tensorflow/generator/` directory allow for simple streaming of data from the MapTextSynthesizer into a Tensorflow program using `from_generator`.

`textsynthinterface.cpp` acts as C wrapper for the MapTextSynthesizer C++ code, allowing it to be wrapped with Python using Ctypes. `data_synth.py` uses Ctypes to make the Python interface of MapTextSynthesize, and `maptextsynth.py` uses the Python-wrapped-C-wrapped-C++ code to generate samples that can be sent to a TensorFlow program.

`data_synth.py` includes the `data_generator` function, which can be used directly as a generator function from which to derive a `tf.data.Dataset` object using `tf.data.Dataset.from_generator`.

Example of this use case can be seen in the `get_dataset` function of `maptextsynth.py`.

`maptextsynth.py` is structured specifically for use by `pipeline.py` in the repository [weinman/cnn_lstm_ctc_ocr](https://github.com/weinman/cnn_lstm_ctc_ocr/tree/full_integration).

#### Notes on Threadability

At the time of writing this (2018) Pangocairo is not thread-safe; following from that, MapTextSynthesizer is not strictly thread-safe. To resolve this, locks were added to avoid race conditions. However, this significantly slows threaded running of the synthesizer; diminishing the prospective production rate.
To circumvent the issues with multi-threading, we suggest using a multi-process technique instead, if you are so inclined.

#### Previous work on this project

If you are interested in seeing the development history of the majority of the features in this project, it can be found at [niehusst/opencv_contrib](https://github.com/niehusst/opencv_contrib/tree/dev). 
