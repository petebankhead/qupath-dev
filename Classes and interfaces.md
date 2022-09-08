# QuPath classes & interfaces

This doc gives an overview of important classes and interfaces in QuPath, along with frank and critical summaries of some of the good and bad parts of their design.

The purposes are to:

1. Help others understand QuPath's code, both for scripting and to write extensions
2. Set the scene for discussions on QuPath's long term future and (re)design


## Getting started: Scripting classes

Two helper classes are used for scripting.

Knowing they exist from the beginning will help, because then you can use QuPath's script editor to access and explore all the other classes.

### [QP](https://qupath.github.io/javadoc/docs/qupath/lib/scripting/QP.html)

`QP` is the main helper class.
It mostly contains a lot of static methods that are generally useful.

For example, in a Groovy script you could access the current `ImageData` and print all its properties and methods using:
```groovy
import qupath.lib.scripting.QP

var imageData = QP.getCurrentImageData()

println QP.describe(imageData)
```

Alternatively, you can import everything from `QP` into the current namespace and then avoid needing to add `QP.` at the start:
```groovy
import static qupath.lib.scripting.QP.*

var imageData = getCurrentImageData()

println describe(imageData)
```

In fact, QuPath's script editor will do this behind the scenes if *Run &rarr; Include default imports* is selected - so you don't need to include the `import` statement separately.

> What `QP.getCurrentImageData()` actually does can be a bit surprising: it accesses an `ImageData` that was set by QuPath before the script was run *for the thread running the script*.
See [this discussion](https://forum.image.sc/t/create-a-shortcut-button-to-run-a-script/71195/9) for more information.

#### The good
* `QP` with static import makes Groovy scripting feel more like using a macro language - and so is hopefully less intimidating for non-coders
  * e.g. `QuPathGUI.getInstance().getImageData().getHierarchy().addPathObjects(pathObject)` becomes instead
  `addObjects(pathObjects)`
* `QP` brings together functionality from lots of different classes, avoiding the need to adding lots of separate `import` statements

#### The bad
* Relying on default imports means that scripts aren't self-contained... and won't run for a user who *doesn't* have *Run &rarr; Include default imports* selected
* `QP` doesn't (and couldn't) contain everything


### [QPEx](https://qupath.github.io/javadoc/docs/qupath/lib/gui/scripting/QPEx.html)

`QP` is defined in QuPath's core processing module: this means it has no access to JavaFX or anything else in the UI.

`QPEx` is a subclass of `QP` that is defined in the UI module, and can therefore access more stuff.
This includes the `QuPathGUI` itself.

```groovy
import static qupath.lib.gui.scripting.QPEx.*

var imageData = getCurrentImageData() // Still works, because QPEx is a subclass
var qupath = getQuPath() // Method not available in QP

println describe(qupath)
```

#### The good
* So long as only methods from `QP` used, it's possible to write scripts that run with or without access to the UI. It doesn't really matter if you import `QP` or `QPEx`.

#### The bad
* Because you can't override a static method in Java, the fact that `QPEx` is a subclass of `QP` can potentially become confusing... especially with static imports involved.
  * I *think* this isn't currently a problem because there are no methods in `QPEx` with the same signature as in `QP` (but I could be wrong... and it certainly wasn't always the case)


## Core classes: Image data & pixels

These classes are the main ones you need to know about when working with QuPath.

### [ImageData](https://qupath.github.io/javadoc/docs/qupath/lib/images/ImageData.html)

This is the starting point when working with an individual image.
Everything else (pixels, metadata, objects) can be extracted from this.

#### The good
* The class is fairly simple & easy to use
* An `ImageData` can easily be saved and reloaded (using the file system directly with [`PathIO`](https://qupath.github.io/javadoc/docs/qupath/lib/io/PathIO.html#writeImageData(java.io.File,qupath.lib.images.ImageData)), or via a project)

#### The bad
* Saved using Java serialization (as a *.qpdata* file). This is a fairly fast & efficiently binary format, but means
  * an `ImageData` object can't be easily read anywhere else (e.g. in Python)
  * care is needed when changing any serialized classes, since it's easy to break compatibility
  * serialization can be a potential security risk when unknown classes might be deserialized
    * [serialization filters](https://github.com/qupath/qupath/commit/1f54e25cdbccf845a7d231b07e9d06246e3feeef) were added to mitigate this
* Uses generics, so we pretty much always have `ImageData<BufferedImage>` - and the `<BufferedImage>` doesn't seem to help much *(see below)*
  * `BufferedImage` defines how the pixels are returned from the `ImageServer` wrapped up in the `ImageData`

It is strongly desirable to move away from serialization in the future.

> #### Why all the `<BufferedImage>`?
> 
> QuPath was originally written using Java Swing, working primarily with RGB images.
> 
> Painting efficiently with Swing (pretty much) required having a [`java.awt.Image`](https://docs.oracle.com/en/java/javase/17/docs/api/java.desktop/java/awt/Image.html), and the most usable subclass of that is `java.awt.image.BufferedImage`.
> 
> However, the `BufferedImage` class isn't available on all platforms (e.g. Android) and isn't very suitable for analysis.
> And, at the time, I thought that might matter.
> 
> Then QuPath switched to using JavaFX, where painting required a [`javafx.scene.image.Image`](https://openjfx.io/javadoc/17/javafx.graphics/javafx/scene/image/Image.html).
> But the JavaFX `Image` class was nowhere near as flexible as `BufferedImage` and wasn't suitable to be our main image representation.
> Fortunately, it was easy to convert a `BufferedImage` to an `Image` via [`SwingFXUtils`](https://openjfx.io/javadoc/17/javafx.swing/javafx/embed/swing/SwingFXUtils.html).
> 
> My thinking was that QuPath should primarily use `BufferedImage`, but should also support other image representations as needed - and thereby avoid locking us into `BufferedImage` forever.
> And so `ImageData<T>` took a generic parameter `<T>`.
> 
> But then as the software evolved, `BufferedImage` reappeared in so many places that there really isn't any occasion when the `T` in `ImageData<T>` stands for anything other than `BufferedImage`.
> And, if we need something else, we start from `BufferedImage` and convert (e.g. using `IJTools` to create an `ImagePlus`, or `OpenCVTools` to create a `Mat`).
> 
> This suggests that the generic parameter **could** be removed and we could just embrace `BufferedImage` fully.
> Although that's tempting, and would simplify some things, I'm not convinced it's the best long-term strategy.
> 
> The main reason is that a `BufferedImage` is inherently 2D and support for multiple channels isn't great.
> There is good reason to expect that QuPath's future involves more multidimensional and multiplexed images.
> 
> Therefore [**ImgLib2**](https://imagej.net/libs/imglib2/) might be a better core library to use. 
> This would require a fairly **major** redesign, and care would need to be taken to retain good performance for the viewer without needing to cache all the image tiles twice.
> However, it has several advantages: 
> * ImgLib2 is very generic
> * Lots of fantastic, open-source [ImgLib2 algorithms](https://github.com/imglib/imglib2-algorithm) are available for processing
> * Potentially better interoperability with [Fiji](https://fiji.sc) / [BigDataViewer](https://imagej.net/plugins/bdv/)
> 
> So the **tentative long term plan is to rewrite pretty much everything in QuPath that uses `BufferedImage`!**
> But that will take considerable effort and planning.


### [ImageServer](https://qupath.github.io/javadoc/docs/qupath/lib/images/servers/ImageServer.html)

The main interface used when requesting pixels and metadata from an image.

The usual way to create one from scratch is via
```java
ImageServer<BufferedImage> server = ImageServers.buildServer(
                                       uri,
                                       args // Optional String array
                                       );
```
Although often that is taken care of internally, and you just grab it from the current `ImageData`:
```java
ImageServer<BufferedImage> server = imageData.getServer();
```

To get the pixel size, ask for a [`PixelCalibration`](https://qupath.github.io/javadoc/docs/qupath/lib/images/servers/PixelCalibration.html) object first:
```java
PixelCalibration cal = server.getPixelCalibration();
```


To get pixels, use
```java
RegionRequest request = RegionRequest.createInstance(
                          server.getPath(),
                          downsample, // double
                          x, y, width, height, z, t // ints
                          );
BufferedImage img = server.readBufferedImage(request);
```


#### The good
* Fairly easy to use for common things
* Supports different implementations via subclassing, which can be auto-discovered using Java services and [`ImageServerBuilder`](https://qupath.github.io/javadoc/docs/qupath/lib/images/servers/ImageServerBuilder.html)
  * This enables QuPath to easily switch between OpenSlide, Bio-Formats, ImageJ and (potentially) other ways of reading images
* Not restricted to reading pixels from a file. Subclasses can read from some other server, or perform calculations before returning pixels (e.g. pixel classification, channel concatenation).
* JSON-serializable, via [`ServerBuilder`](https://qupath.github.io/javadoc/docs/qupath/lib/images/servers/ImageServerBuilder.ServerBuilder.html), so can be used in a project.

#### The bad
* Naming isn't great
  * Image 'reader' would arguably be better than 'server'
  * `server.getPath()` is badly named, since it now returns an ID - not actually a file path. The ID can be anything, so long as it's unique for a server. The reason is that a file path isn't necessarily enough to uniquely identify an image, and some images (e.g. from a pixel classification) don't *have* a file path because pixels are generated on demand.
  * [`ServerBuilder`](https://qupath.github.io/javadoc/docs/qupath/lib/images/servers/ImageServerBuilder.ServerBuilder.html) is also badly named, because it solves a different problem (JSON serialization) from [`ImageServerBuilder`](https://qupath.github.io/javadoc/docs/qupath/lib/images/servers/ImageServerBuilder.html) (creating an `ImageServer` from a URI and args).


### [RegionRequest](https://qupath.github.io/javadoc/docs/qupath/lib/regions/RegionRequest.html)

Defines a 2D bounding box and downsample that is used to request pixels from an `ImageServer`.

The `RegionRequest` also stores the server `path` (really a unique ID), although this doesn't *necessarily* have to be correct when requesting pixels because the server knows its own ID.
However, it's important because `RegionRequest`s are used as keys when caching pixels.

The code above shows one way to create a `RegionRequest`.
Others include

```java

RegionRequest requestForWholeImage = RegionRequest.createInstance(
                          server
                          // Assume first plane (z=0, t=0)
                          );
                          
RegionRequest requestForDownsampledWholeImage = RegionRequest.createInstance(
                          server,
                          downsample // double
                          // Assume first plane (z=0, t=0)
                          );
                          
RegionRequest requestForROI = RegionRequest.createInstance(
                          server,
                          downsample, // double
                          roi // Bounding box taken from ROI
                          );
```

#### The good
* A `RegionRequest` is immutable, so can be (and is) used as a key in a `Map`
* A `RegionRequest` can store **any** downsample value (> 0... although typically also >= 1 since `ImageServer` implementations don't necessarily support upsampling).
This isn't limited by the resolution levels available in the pyramidal image.
Consequently, the consumer doesn't need to care what resolution levels are present: it's up to the `ImageServer` to figure out how to read the corresponding pixels and rescale if necessary.
* The fact that the bounding box is **always for the full-resolution image** (i.e. `downsample=1`) simplifies things.
* A `RegionRequest` can be reused, possibly with a call to `RegionRequest.updatePath(path)` to match it to another image. This is handy when exporting corresponding regions from multiple images (e.g. original pixels and a binary mask).

#### The bad
* Feels awkward in simple cases to create a `RegionRequest` every time pixels are required
* We don't always know the size of the output image when `downsample != 1`, because we aren't sure how the `ImageServer` will handle any necessary rounding when mapping the requested coordinates to the actual resolutions available in the image.
* Originally (as far as I can remember...), any `RegionRequest` might be cached. But that can be inefficient. The same pixels might often need to be read from disk to fulfil different region requests, because the bounding box is slightly shifted or the downsample isn't quite the same.

An example of the last problem: requesting the same bounding box using `downsample=1` and `downsample=1.2` would give different `RegionRequest` objects (and therefore cache keys) but almost certainly require accessing the same pixels.
Ideally, we'd just cache at `downsample=1` since that's what we read from disk.


### [TileRequest](https://qupath.github.io/javadoc/docs/qupath/lib/images/servers/TileRequest.html)

The `TileRequest` class was introduced to overcome some of the efficiency problems and ambiguities of `RegionRequest`.
It's primarily used internally by `ImageServer`s.

A `TileRequest` operates like a `RegionRequest`, except that the bounding box coordinates are given at a specific resolution that is returned by [`ImageServer.getPreferredDownsamples()`](https://qupath.github.io/javadoc/docs/qupath/lib/images/servers/ImageServer.html#getPreferredDownsamples()).
A `RegionRequest` can then be computed automatically from a `TileRequest` by multiplying the bounding box coordinates by the downsample.

`TileRequest` is central to the operation of [`AbstractTileableImageServer`](https://qupath.github.io/javadoc/docs/qupath/lib/images/servers/AbstractTileableImageServer.html), which was added for QuPath v0.2.
This breaks up any arbitrary `RegionRequest` into the tiles needed to fulfil it, caches tiles as needed, and standardizes any resizing required to match the desired downsample.
 
Subclasses of `AbstractTileableImageServer` therefore need shouldn't implement `readBufferedImage(RegionRequest)` but rather only the abstract [`readTile(TileRequest)`](https://qupath.github.io/javadoc/docs/qupath/lib/images/servers/AbstractTileableImageServer.html#readTile(qupath.lib.images.servers.TileRequest)) method.

This makes adding a new `ImageServer` implementation much easier.


#### The good
* Introducing `TileRequest` was able to dramatically reduce the number of duplicate requests for the same pixels from storage, and therefore make some things much more efficient.
* The user (including coders) almost never needs to care about the existence of `TileRequest` because it is used internally.

#### The bad
* *Sometimes* a user would like to work with `TileRequest` directly, because they want to control the size of the image returned for an arbitrary downsample. That's currently not easy to do.


### [ImageRegion](https://qupath.github.io/javadoc/docs/qupath/lib/regions/ImageRegion.html)

Like `RegionRequest`, but doesn't include a path or downsample value.



### [ImageType](https://qupath.github.io/javadoc/docs/qupath/lib/images/ImageData.ImageType.html)

Enum that represents the type of the image, in a way that is relevant for some commands.

Its main use is to distinguish between fluorescence and brightfield images (with different stains).
Cell detection needs this to know whether it's looking for nuclei containing higher pixel values or lower pixel values than the surrounding background.

#### The good
* Simple
* Switching a brightfield type (e.g. H&E to H-DAB) and back still preserves the stain vectors
  * i.e. switching the image type doesn't irretrievably lose any custom stain vectors

#### The bad
* Types are nowhere near exhaustive... many more types are needed for images that might be used with QuPath
* 'Brightfield (Other)' covers a multitude of possible stain combinations
  * We probably don't want to add a load more possible types for all of them
* Setting the type is a pain for users; auto-estimating is error-prone


### [ColorDeconvolutionStains](https://qupath.github.io/javadoc/docs/qupath/lib/color/ColorDeconvolutionStains.html)

Represents the stain vectors used for stain separation, using Ruifrok & Johnston's color deconvolution method.

Used only if the `ImageType` is brightfield.

#### The good
* Makes color deconvolution easy to apply in common cases
* Can be set in a one-line Groovy script

#### The bad
* Only relevant for a subset of images (mostly brightfield histology)
* Inherently limited to 2 or 3 chromogenic stains
* Other methods of stain separation are available
* Presupposes the color deconvolution model makes sense... but if the 'original' pixels have been transformed in some way (e.g. gamma correction, ICC profile) then perhaps it doesn't
* Optimizing stain vectors is tricky - strong possibility people just stick to the defaults
* Uses color, but doesn't normalize for stain intensity


### [Workflow](https://qupath.github.io/javadoc/docs/qupath/lib/plugins/workflow/Workflow.html)

Used to log specific processing steps, which can then be converted into scripts.

It doesn't log *everything*, primarily because some things are very image-specific (e.g. drawing an annotation)... although perhaps also because some commands might simply not have registered a `WorkflowStep` even with they potentially should.

#### The good
* Stores *most* key steps for *most* common analysis (important for traceability)
* Can be used to generate batch scripts

#### The bad
* Doesn't store everything: key steps can be missing (e.g. if the user ran a script)
* Batch scripts generated from the workflow often need manual editing to work
* Making new commands workflow-friendly takes extra effort


## Core classes: Objects, ROIs, classifications & measurements

Analysis in QuPath generally involves generating and querying objects.
The idea is explained in the [Objects](https://qupath.readthedocs.io/en/stable/docs/concepts/objects.html) page of the documentation.

A key design decision is that how objects are represented should be kept separated from how they are displayed.

The following classes mostly achieve this... except for the occasional appearance of color properties.

> #### Why is color usually an `Integer`?
> 
> Core QuPath classes that have color properties (`PathObject` and `PathClass`) represent color as `Integer`, which uses a packed (A)RGB format.
> The reason is that I wanted to 
> * avoid relying on a specific class like `java.awt.Color` or `javafx.scene.paint.Color`
> * maximize efficiency by avoiding the use of a custom object with multiple fields, while still allowing the color to be missing (which is the reason for `Integer` and not `int`)
> 
> Like many things, I don't know if this was the right decision.
> 
> In any case, `ColorTools`, `ColorToolsAwt` and `ColorToolsFX` are useful classes when working with colors as integers and converting them to other forms.


### [PathObject](https://qupath.github.io/javadoc/docs/qupath/lib/objects/PathObject.html)

This is the base class for representing 'things' in an image, defined with a region of interest (ROI) and potentially having classifications and measurements associated.
This is explained further on the [Objects](https://qupath.readthedocs.io/en/stable/docs/concepts/objects.html) page.

`PathObject` itself is an abstract class.
Useful subclasses include
* `PathRootObject` (one per image, doesn't have a ROI)
* `PathAnnotationObject` (for big things)
* `PathDetectionObject` (for smaller things)
* `PathTileObject` (for square tiles & superpixels)
* `PathCellObject` (specifically for cells with a boundary and nucleus)
* `TMACoreObject` (specifically for tissue microarray cores)

#### The good
* Simple, flexible and fairly efficient - it seems to serve its job well
* Maps pretty well to a GeoJSON [Feature object](https://www.rfc-editor.org/rfc/rfc7946#section-3.2) - and would be easy to reimplement in other software

#### The bad
* I'm not convinced the use of subclasses (`PathAnnotationObject`, `PathDetectionObject` etc.) is a good idea, or if it just adds unnecessary complexity.
  * In the case of `PathCellObject` perhaps it is (because cells support extra nucleus ROIs)... but even then I don't know if it's the right approach.
    * A nucleus should perhaps even be a distinct object in its own right, not only a ROI. Then the cell boundary and the nucleus could simply be detections... although they tend to share a classification, so that becomes complicated.
  * Subclassing allows nasty things like creating [`ParallelTileObject`](https://qupath.github.io/javadoc/docs/qupath/lib/plugins/ParallelTileObject.html) as a trick to handle temporary tiles when processing a large region
    * I don't have a better way to do this, but it feels like a hack
  * The use of `instanceof` checks is ugly; `PathObject.isAnnotation()` etc. seems to me nicer than `pathobject instanceof PathAnnotationObject`
* `TMACoreObject` is another outlier because it only really supports ellipse ROIs - but sometimes TMA cores are close to one another and perhaps supporting arbitrary ROIs would be useful


### [PathClass](https://qupath.github.io/javadoc/docs/qupath/lib/objects/classes/PathClass.html)

Simple class representing a classification.
For more info, see [Classifications](https://qupath.readthedocs.io/en/stable/docs/concepts/classifications.html).

#### The good
* Each `PathClass` should be a singleton. This makes it easy to check if objects have exactly the same classification, e.g. `pathClass1 == pathClass2`.
* Immutable...ish. The name and string representation of the `PathClass` can't be changed, but the color can be.

#### The bad
* Using the colon `:` as a separator has proven problematic for using `PathClass` with some ontologies
* Need to remember to ensure each `PathClass` is a singleton using [`PathClassFactory`](https://qupath.github.io/javadoc/docs/qupath/lib/objects/classes/PathClassFactory.html)
* *Sometimes* it may be desirable to support multiple classifications for an object. Currently that can only be achieved using subclasses.
  * See [Multiplexed analysis](https://qupath.readthedocs.io/en/stable/docs/tutorials/multiplex_analysis.html) for one way to work with that
  
#### The in-between
* `PathClass` replies a bit on hacks to overcome other limitations... although they seem to (more or less) work.
  * [Intensity classifications](https://qupath.readthedocs.io/en/stable/docs/concepts/classifications.html?#intensity-classifications) are a bit special, used for summary measurements
  * Adding an asterisk to the name creates an [ignored classification](https://qupath.readthedocs.io/en/stable/docs/concepts/classifications.html?#ignored-classifications)
    * Off-hand, I couldn't tell you how that interacts with subclassifications...


### [ROI](https://qupath.github.io/javadoc/docs/qupath/lib/roi/interfaces/ROI.html)

Interface defining how a region of interest is represented.

Implementing classes handle specific kinds of ROI, including rectangles, polygons... and the dreaded ellipses.

The original goal in v0.1.2 and before was to have a simple, serializable ROI class with minimal overhead, which could be converted to a [`java.awt.Shape`](https://docs.oracle.com/en/java/javase/17/docs/api/java.desktop/java/awt/Shape.html) rapidly for painting.

Complex ROIs could be represented and manipulated using [`java.awt.geom.Area`](https://docs.oracle.com/en/java/javase/17/docs/api/java.desktop/java/awt/geom/Area.html).

This worked reasonably, but then a lot of extra code was needed to do interesting stuff for analysis.

Since [*Java Topology Suite* (JTS) is really the standard for that kind of thing](https://en.wikipedia.org/wiki/JTS_Topology_Suite), ROIs were then changed to support easy conversion to a JTS [`Geometry`](https://locationtech.github.io/jts/javadoc/) (with `ROI.getGeometry()` available).

This made a lot of extra functionality possible, including making measurements, combining objects, calculating distances between objects, converting to GeoJSON and so on.

*However*, JTS has a much stricter idea of polygons and multipolygons - which means that a lot of extra code needed to be added to try to ensure ROIs are converted to valid JTS geometries.

For example, a QuPath `PolygonROI` can be drawn interactively and potentially have self-intersections (e.g. a bowtie shape) - but this can't be a single JTS `Polygon`.
And JTS doesn't support ellipses, so there's no direct counterpart for `EllipseROI`.

For this reason, we can't just use `Geometry` and discard `ROI` - we do need to switch between them.
The [`GeometryTools`](https://qupath.github.io/javadoc/docs/qupath/lib/roi/GeometryTools.html) class was developed to try to help.

#### The good
* Fairly simple and self contains - does not include properties about how the ROI is displayed (e.g. line thickness, color)
* *Most* ROIs can easily be converted to JTS `Geometry` objects, and also to GeoJSON

#### The bad
* Limited to 2D
* Some operations rely on being able to convert to a JTS `Geometry`, but this potentially imposes some limitations. For example, ellipses and curves can't be fully supported (they need to be 'polygonized' first).


### [MeasurementList](https://qupath.github.io/javadoc/docs/qupath/lib/measurements/MeasurementList.html)


### [PathObjectHierarchy](https://qupath.github.io/javadoc/docs/qupath/lib/objects/hierarchy/PathObjectHierarchy.html)



## Core classes: Projects


### [Project](https://qupath.github.io/javadoc/docs/qupath/lib/projects/Project.html)

#### The good
* Uses JSON for serialization/deserialization (so can potentially be read or written elsewhere)
* It's an interface, so can potentially be backed by some other implementation (i.e. isn't limited to being on the local file system)

#### The bad


### [ProjectImageEntry](https://qupath.github.io/javadoc/docs/qupath/lib/projects/ProjectImageEntry.html)



## Core classes: Classifiers

### Object classifier

### Pixel classifier



## Processing classes

### PathPlugin

### ParameterList

### IJTools

### OpenCVTools

### ImageOps


## UI classes

### QuPathGUI

### QuPathExtension

### PathOverlay

