---
layout: post
title: Unsupervised flood mapping
---

Floods can have a [catastrophic](http://fortune.com/2016/08/25/top-ten-fema-funded-disasters/) [impact](https://www.washingtonpost.com/news/capital-weather-gang/wp/2016/10/14/a-flood-disaster-in-n-c-satellite-photos-before-and-after-hurricane-matthew/?utm_term=.e549c201ae1a) on local communities and their [economies](https://weather.com/news/news/hurricane-matthew-north-carolina-update), resulting in loss of human life and property, damage in [agriculture](http://www.cnbc.com/2016/06/29/flooding-in-west-virginia-impacts-states-agriculture.html) and [destruction of infrastructure](http://www.dailycamera.com/news/boulder-flood/ci_24163947/report-more-than-91m-flood-damage-across-unincorporated). Accurate and near real-time derived flood maps are of paramount importance in directing search and rescue operations and relief efforts.
We have developed a practical and scalable solution to flood mapping in the form of an unsupervised flood water detector which utilizes 4- or 8-band DigitalGlobe imagery and can process an entire strip in a matter of minutes on a default GBDX instance.   

<br>
![baton-rouge.png]({{ site.baseurl }}/images/flood-water/foothills.png)
*Section of WorldView-2 image captured over the Longmont, CO on 09/17/2013, a few days after heavy rains swept across the area (catalog id 1030010027BFBF00). Flood water is shown in dark red. See [here]({{ site.baseurl }}/pages/flood-water/foothills.html) for the full-page map.*
<br><br>


## Floods

A [flood](https://en.wikipedia.org/wiki/Flood) is technically a topography-dependent overflow of water which submerges land that is normally dry. Regions prone to flooding are referred to as **flood zones**. The [Federal Emergency Management Agency (FEMA)](https://www.fema.gov) has manually compiled an extensive [archive](https://msc.fema.gov/portal/search) of flood zones across the US, which are used for floodplain management and insurance rating purposes. Although FEMA flood zones can inform preventative measures in order to reduce flood damage, they do not necessarily delineate the extent of an actual flood.

Flood maps are notoriously hard to obtain yet they are crucial in assessing damage extent, in directing early-stage crisis response, and in providing new data to improve the accuracy of existing flood zones. **Manual** delineation of flooded areas on satellite or aerial imagery by experts is possible but it is laborious, slow and prone to human error. **Supervised**, i.e., semi-automated, detection of flood water involves collecting a limited number of training samples which capture the spectral properties of flood water, and training a classifier to identify flood water pixels. The problem with this approach is that flood water, in contrast to 'clean' water, does not have a distinctive radiometric signature. Its spectral properties depend on the type and concentration of floating particles of man-made and natural materials which have been washed away in the flood. As a result, a supervised algorithm fine-tuned to one area needs to be re-trained before deployment in another area.

## Unsupervised solution

To address these challenges we developed a **fully unsupervised** solution for flood mapping based on the detection of impure water pixels. The detector utilizes atmospherically compensated 4- or 8-band DigitalGlobe imagery and can process an entire strip in a matter of minutes on a r4.2xlarge GBDX instance. Behind the scenes, it employs a pixel based, generic material discriminator that works with stacks of automatically generated, raster semantic layers each imprinting radiometric 'trends'. The discriminator evaluates the contribution of each layer in the semantic data space to produce a likelihood estimate for each pixel being member of a water-saturated body. The solution is available as a **protocol** in Protogen, the in-house image analytics software suite of GBDX, and is a member of the Protogen crisis management toolbox.

By default, the detector identifies all water-saturated bodies, including mud. The minimum moisture content can be specified by the user in the form of a ```moisture``` percentage. In addition, the detector includes a [structural opening operator](https://en.wikipedia.org/wiki/Mathematical_morphology#Opening) that uses disk-shaped kernels to cut off linear segments of width less than the ```min_width``` threshold and a size filter that removes objects of surface area less than the ```min_size``` threshold. These structural filters can be used to minimize noise from paved roads and tar-coated roof shingles, which contain materials that the detector is sensitive to.

Following is a Python example that shows how to run the protocol on the 8-band image ```example.tif```.
```min_width``` and ```min_size``` are set in order to remove noise from typical roads and house roofs.

```python
import protogen
p = protogen.Interface('crime', 'flood_water_mapper')	# 'crime' as in CRIsis ManagemEnt !
p.crime.flood_water_mapper.moisture  = 1        # minimum moisture content as a percentage
p.crime.flood_water_mapper.min_width = 10       # road width in meters
p.crime.flood_water_mapper.min_size = 1000      # average house roof size in square meters
p.crime.flood_water_mapper.binary_output = True # output format
p.image = 'example.tif'
p.image_config.bands = [1, 2, 3, 4, 5, 6, 7, 8] # use 8 bands
p.execute()
```


## Deployment

We recently deployed the flood mapper in [Madagascar](http://gbdxstories.digitalglobe.com/enawo-madagascar) in order to assess the impact of hurricane Enawo. For comparison purposes, we downloaded [TerraSAR-X data](http://www.unitar.org/unosat/node/44/2554?utm_source=unosat-unitar&utm_medium=rss&utm_campaign=maps) from the United Nations Institute for Training and Research (UNITAR) and redid the map for the Maroantsetra region.

<br>
{% raw %}
<iframe frameborder="no" border="0" marginwidth="0" marginheight="0" width="800" height="800" src="../pages/flood-water/maroantsetra.html"></iframe>
{% endraw %}

*Maroantsetra, Madagascar, 3/11/17 (GeoEye-1 image with catalog id 1050010008989C00). Flood water derived with Protogen in brown. SAR water detection on 03/08 and 03/10 in green and blue, respectively (source: UNITAR). Click on each tab to activate the corresponding layer. Full page view [here]({{ site.baseurl }}/pages/flood-water/maroantsetra.html).*
<br><br>

The overall observation is that a substantial portion of our flood map is not identified by SAR. This is most likely due to the fact that the water saturated bodies identified by our detector, as opposed to liquid water, do not act as specular reflectors. Moreover, in urban areas, flooded ground in front of the wall of a building viewed in the range direction may be [misclassified as dry]((http://centaur.reading.ac.uk/24935/1/TGRS-2010-00649.R2_all.pdf)) due to the strong reflection.
More investigation is needed to determine the validity of these interpretations.    

We also used the flood mapper on imagery captured at three recent important flooding events in the US;
the [Colorado](https://en.wikipedia.org/wiki/2013_Colorado_floods) floods on September 2013, the [Louisiana](https://en.wikipedia.org/wiki/2016_Louisiana_floods) floods on August 2016, and the North Carolina floods on October 2016 due to [hurricane Matthew](https://en.wikipedia.org/wiki/Hurricane_Matthew). All three events were declared  [major disasters](https://www.fema.gov/disasters) by FEMA.

You can find our code [here](https://github.com/PlatformStories/notebooks/blob/master/Unsupervised%20flood%20mapping.ipynb).
For each catalog id, the flood mask is computed from the multispectral image, and both the mask and the compressed pansharpened image are uploaded to Mapbox which generates the tilesets used in the maps.

Sample results are shown in the maps below. Each map consists of three layers; the pansharpened image, the flood map in brown, and the FEMA flood zones, directly from the FEMA [WMS service](https://hazards.fema.gov/femaportal/wps/portal/NFHLWMS), which you can interpret using the legend. (Note that as you zoom out, the FEMA layer disappears.)  

<br>
![fema-legend.png]({{ site.baseurl }}/images/flood-water/fema-legend.png)

*Explanation of flood zones (courtesy of FEMA).*
<br><br>

<br>
{% raw %}
<iframe frameborder="no" border="0" marginwidth="0" marginheight="0" width="800" height="800" src="../pages/flood-water/foothills.html"></iframe>
{% endraw %}

*Longmont, Colorado, 9/17/2013 (WorldView-2 image with catalog id 1030010027BFBF00). Flood water in brown. FEMA colors are explained in the legend. Click on each tab to activate the corresponding layer. Full page view [here]({{ site.baseurl }}/pages/flood-water/foothills.html).*
<br><br>

<br>
{% raw %}
<iframe frameborder="no" border="0" marginwidth="0" marginheight="0" width="800" height="800" src="../pages/flood-water/baton-rouge.html"></iframe>
{% endraw %}

*Baton Rouge, Louisiana, 8/16/2016 (WorldView-2 image with catalog id 103001005B38CE00). Flood water in brown. FEMA colors are explained in the legend. Click on each tab to activate the corresponding layer. Full page view [here]({{ site.baseurl }}/pages/flood-water/baton-rouge.html).*
<br><br>

<br>
{% raw %}
<iframe frameborder="no" border="0" marginwidth="0" marginheight="0" width="800" height="800" src="../pages/flood-water/north-carolina.html"></iframe>
{% endraw %}

*Tarboro, North Carolina, 10/13/2016 (WorldView-3 image with catalog id 10400100238BDE00). Flood water in brown. FEMA colors are explained in the legend. Click on each tab to activate the corresponding layer. Full page view [here]({{ site.baseurl }}/pages/flood-water/north-carolina.html).*
<br><br>

Our flood maps coincide to a large extent with the designated flood zones by FEMA. Brown areas outside the FEMA flood zones are likely flooded basins, existing water bodies that are contaminated by flood water, or generally water-saturated bodies of all sizes. Areas where flood water is expected but is not detected are due to coverage from dense clouds or vegetation. In these cases, SAR data can provide an asset.

False positives due to roads and roof materials are minimal thanks to the size and width filters. Some road segments remain; that can be either because the width filter does not remove them or because they are actually partially or fully covered by water.

Overall, our unsupervised method appears to perform very well in the variety of scenarios where we tested it. Its speed and flexibility make it a promising solution for near real-time flood mapping. [Get in touch](mailto:kostas.stamatiou@digitalglobe.com) if you are interested in finding out more about this technology or other capabilities on GBDX.
