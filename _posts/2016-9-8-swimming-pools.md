---
layout: post
title: Finding swimming pools in Australia
---

Information about built environments is extremely valuable
to [insurance companies](https://www.allstate.com/tools-and-resources/home-insurance/swimming-pool-insurance.aspx), [tax assessors](http://www.spiegel.de/international/europe/finding-swimming-pools-with-google-earth-greek-government-hauls-in-billions-in-back-taxes-a-709703.html), and public agencies, as it empowers a wide range of decision-making including
urban and regional planning and management, risk estimation, and emergency response.
Extracting this information using human analysts to scour satellite imagery
is prohibitively expensive and time consuming. Feature extraction and machine
learning algorithms are the only viable way to perform this type of attribution
at scale.  

This is why the Australian company PSMA teamed up with DG to develop the product
[Geoscape](https://www.psma.com.au/geoscape): a diverse set of building attributes
including height, rooftop material, solar panel installation and presence of a swimming pool
in the property across the **entire Australian continent**.

We'll demonstrate how we used deep learning on GBDX to identify swimming pools in thousands of properties across
[Adelaide](https://en.wikipedia.org/wiki/Adelaide), a major city on the southern coast of Australia with a population of approximately one million. The result: **31071 of the 670784 properties** we classified contain pools; approximately 4.6%. This is more or less consistent with the Australian Bureau of Statistics data for [households with a pool in South Australia](http://www.abs.gov.au/ausstats/abs@.nsf/Products/F4641220D19FD971CA2573A80011AD41?opendocument). Compare this with the corresponding figure for New South Wales which is upwards of 10%. Given the similar arid or semi-arid climate, could we identify other reasons for the discrepancy? A little digging into the regions' economic health might provide some clues:
New South Wales has [a significantly higher average annual income](http://www.abs.gov.au/ausstats/abs@.nsf/mf/5673.0.55.003) compared to South Australia. Given the installation and maintenance costs of a swimming pool, their number could be a potential indicator of economic health!

![properties.png]({{ site.baseurl }}/images/swimming-pools/properties.png)
*Property boundaries in Adelaide. Green/red indicate presence/absence of pool.*

## How

Our GBDX workflow is shown in the following figure.

![workflow_diag.png]({{ site.baseurl }}/images/swimming-pools/wf.png)  
*Platform workflow for property classification.*

### Preprocessing

The workflow begins with the file [properties.geojson](https://github.com/PlatformStories/PlatformStories.github.io/blob/master/pages/swimming-pools/properties.geojson). This file contains a collection of polygons in (longitude, latitude) coordinates, each representing a property. Each polygon has two attributes: an image_id, which determines the DG catalog id of the satellite image corresponding to that polygon, and a feature_id, which is simply a number that uniquely identifies that property. In this example, the file only contains properties from a single WV03 image over Adelaide, Australia, with catalog id 1040010014800C00.

The main idea is to label a small percentage of the property parcels using [crowdsourcing](http://www.tomnod.com/) in order to create a training set [train.geojson](https://github.com/PlatformStories/PlatformStories.github.io/blob/master/pages/swimming-pools/train.geojson). We then use train.geojson to train a CNN-based classifier to identify the presence of a swimming pool in each of the remaining unlabeled properties of properties.geojson (referred to as [target.geojson](https://github.com/PlatformStories/PlatformStories.github.io/blob/master/pages/swimming-pools/target.geojson)). For object classification at a continental or global scale this procedure is a must; it would be virtually impossible to label millions of properties manually in a reasonable amount of time.

Before executing the workflow, the raw image has to be ordered from the factory and processed into a format that is viewable and also usable by a machine learning algorithm. The last part is trickier than it sounds. We lovingly refer to it as UGHLi: Undifferentiated Geospatial Heavy Lifting. In this example, it involves orthorectification, atmospheric compensation, pansharpening and dynamic range adjustment; all this can be achieved with a single GBDX task, [AOP_Strip_Processor](http://gbdxdocs.digitalglobe.com/docs/advanced-image-preprocessor). You can explore the image below or click [here]({{ site.baseurl }}/pages/swimming-pools/adelaide.html) for a full page view.

{% raw %}
<iframe frameborder="no" border="0" marginwidth="0" marginheight="0" width="800" height="800" src="../pages/swimming-pools/adelaide.html"></iframe>
{% endraw %}


### Workflow inputs

The workflow requires three inputs.

- A collection of labeled polygons in geojson format (the training data). In this example, the labeled properties are found in
[train.geojson](https://github.com/PlatformStories/PlatformStories.github.io/blob/master/pages/swimming-pools/train.geojson).

  ![train_geojson.png]({{ site.baseurl }}/images/swimming-pools/train_geojson.png)  
  *A sample of labeled properties.*

- A collection of polygons which will be classified, in geojson format (the target data). In this example, the properties to be classified are found in [target.geojson](https://github.com/PlatformStories/PlatformStories.github.io/blob/master/pages/swimming-pools/target.geojson).

  ![target_geojson.png]({{ site.baseurl }}/images/swimming-pools/target_geojson.png)  
  *A sample of properties to be classified.*

- The UGHLi'd image(s) which the polygons in the training and target data overlay, in [GeoTiff](https://en.wikipedia.org/wiki/GeoTIFF) format. In this example, it's the Adelaide image.   

### Tasks

The workflow involves two tasks.

- [train-cnn-classifier](https://github.com/PlatformStories/train-cnn-classifier): Trains a CNN classifier on the polygons in train.geojson. Required inputs are train.geojson, associated image strips, and class names as a string argument. This task returns the architecture and weights of the trained model.

   ![train-cnn-classifier.png]({{ site.baseurl }}/images/swimming-pools/train-cnn-classifier.png)  
   *train-cnn-classifier takes train.geojson and imagery, and produces a trained CNN classifier.*

   You can find more information on the algorithm used in this example
   [here](https://developer.digitalglobe.com/gbdx-poolnet-identifying-pools-satellite-imagery/) and
   [here](https://github.com/DigitalGlobe/mltools/tree/master/examples/polygon_classify_cnn).

- [deploy-cnn-classifier](https://github.com/PlatformStories/deploy-cnn-classifier): Deploys a trained CNN model on target.geojson. Requires a trained model, target.geojson, and associated image strips, and returns classified.geojson. deploy-cnn-classifier can classify approximately 250,000 polygons per hour.

  ![deploy_cnn_classifier.png]({{ site.baseurl }}/images/swimming-pools/deploy_cnn_classifier.png)  
  *deploy-cnn-classifier takes a model, target.geojson and imagery, and produces classified.geojson.*


### Workflow outputs

The workflow has two outputs.

- A [trained keras model](https://keras.io/models/about-keras-models/), which contains the model architecture, the final weights, a test report and the weights after each training epoch:

      /trained_model
              ├── model_arch.json
              ├── model_weights.h5
              ├── test_report.txt
              └── model_weights
                  ├── round_1
                  │   └── weights after each epoch
                  └── round_2
                      └── weights after each epoch  

  More information on the model architecture can be found [here](https://github.com/DigitalGlobe/mltools/tree/master/examples/polygon_classify_cnn).

- The file classified.geojson which includes all the properties in target.geojson classified in 'Swimming pool' and 'No swimming pool'.


## Executing the workflow

We'll now execute the workflow in gbdxtools. Start an iPython terminal, create a GBDX interface, and get the input location information:

```python
from gbdxtools import Interface
from os.path import join
import uuid

gbdx = Interface()

# specify location of files needed in this story
input_location = 's3://gbd-customer-data/58600248-2927-4523-b44b-5fec3d278c09/platform-stories/swimming-pools'
```

Create a train_task object and set the required inputs:

```python
train_task = gbdx.Task('train-cnn-classifier')
train_task.inputs.images = join(input_location, 'images')
train_task.inputs.geojson = join(input_location, 'train-geojson')
train_task.inputs.classes = 'No swimming pool, Swimming pool'     # Classes exactly as they appear in train.geojson
```

In training our model, we can set optional hyper-parameters. See the [docs](https://github.com/PlatformStories/train-cnn-classifier) for detailed information. Training should take around 3 hours to complete.

```python
train_task.inputs.nb_epoch = '30'
train_task.inputs.nb_epoch_2 = '5'
train_task.inputs.train_size = '5000'
train_task.inputs.train_size_2 = '2500'
train_task.inputs.test_size = '1000'
train_task.inputs.bit_depth = '8'         # Provided imagery is dra'd
```  

Create a deploy_task object with the required inputs, and set the *model* input as the output of train_task.

```python
deploy_task = gbdx.Task('deploy-cnn-classifier')
deploy_task.inputs.model = train_task.outputs.trained_model.value     # Trained model from train_task
deploy_task.inputs.images = join(input_location, 'images')
deploy_task.inputs.geojson = join(input_location, 'target-geojson')
```

Specify the classes for the deploy task. We can also restrict the size of polygons that we deploy on and set the appropriate bit depth for the input imagery:

```python
deploy_task.inputs.classes = 'No swimming pool, Swimming pool'
deploy_task.inputs.bit_depth = '8'
deploy_task.inputs.min_side_dim = '10'    # Minimum acceptable side dimension for a polygon
```

String the two tasks together in a workflow and save the output in a directory with
a unique string identifier under bucket/prefix/platform-stories/swimming-pools/trial-runs:

```python
workflow = gbdx.Workflow([train_task, deploy_task])

# set output location to platform-stories/trial-runs/random_str within your bucket/prefix
random_str = str(uuid.uuid4())
output_location = join('platform-stories/trial-runs', random_str)

# save workflow outputs
workflow.savedata(train_task.outputs.trained_model, join(output_location, 'trained-model'))
workflow.savedata(deploy_task.outputs.classified_geojson, join(output_location, 'classified-geojson'))
```

Execute the workflow:

```python
workflow.execute()
```

Depending on the hyper-parameters set on the model, training sizes, and size of the deploy file, this workflow can take several hours to run. You may check on the status periodically with the following commands:

```python
workflow.status
workflow.events # a more in-depth summary of the workflow status
```

You can download your outputs as follows.

```python
# train-cnn-classifier sample output: final model
gbdx.s3.download(join(output_location, 'trained-model/model_architecture.json'), 'trained-model/')
gbdx.s3.download(join(output_location, 'trained-model/model_weights.h5'), 'trained-model/')
gbdx.s3.download(join(output_location, 'trained-model/test_report.txt'), 'trained-model/')

# deploy-cnn-classifier sample output
gbdx.s3.download(join(output_location, 'classified-geojson'), 'classified-geojson')
```

## Visualizing the results

We [uploaded](https://github.com/PlatformStories/upload-to-mapbox) the Adelaide image and classified.geojson to our Mapbox account, and then used the [Mapbox GL Javascript library](https://www.mapbox.com/mapbox-gl-js/api/) to display the raster and vector tilesets. Here are the results (full page view [here]({{ site.baseurl }}/pages/swimming-pools/adelaide-classified-properties.html)). Green/red polygons indicate presence/absence of pool.

{% raw %}
<iframe frameborder="no" border="0" marginwidth="0" marginheight="0" width="800" height="800" src="../pages/swimming-pools/adelaide-classified-properties.html"></iframe>
{% endraw %}  


## Discussion

This is a small example of what is possible on GBDX. Once a trained model is
obtained, it can be deployed on properties over hundreds or thousands of different images
**in parallel**. Continental scale classification becomes a matter of hours.

![scale.png]({{ site.baseurl }}/images/swimming-pools/scale.png)
*GBDX allows large scale parallelization.*  

In order to exploit the power of GBDX, an algorithm must be packaged into a GBDX task.
The procedure is described [here](https://platformstories.github.io/create-task) in detail.
Get after it!
