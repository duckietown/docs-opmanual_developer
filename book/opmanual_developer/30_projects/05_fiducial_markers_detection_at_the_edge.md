# Projects: Fiducial Markers Detection at the Edge {#sec:projects_fiducial level=sec status=ready}

Author: Alexander Hatteland

Maintainer: Alexander Hatteland

This section will introduce the Fiducial Markers that the
Duckietown community employs in all the Autolabs. It also explains what
markers are used for, and how we are extracting the necessary information from them.
This section also suggests a more decentralized apporach to the online localization 
solution of the Autolab. 

See also: To get a detailed overview of the tests conducted: [Onboard apriltag processing](https://ethidsc.atlassian.net/wiki/spaces/DS/pages/465371137/Design+Document+The+on-board+Apriltag+Processing?)

<minitoc/>

## Motivation {#motication-for-project}
With an accurate localization system, we are finally able to provide Autonomous Mobility on
Demand to all the users of the Autolab. The online localization can be used to a variety of 
different demos and as a help tool for AIDO submission evaluations. In this project we aim to improve 
the fiducial detection, to have a faster, and more decentralized localization system.

We envision a fiducial marker detection node that can be processed on each Watchtower to
reduce the load of the processing on one device (the town/server), and from there move to a more decentralized
system. The way the old architecture worked can be found 
[here](https://docs.duckietown.org/daffy/downloads/opmanual_autolab/docs-opmanual_autolab/builds/211/opmanual_autolab/out/autolab_localization_software.html).

The watchtowers need to extract the relative poses of each fiducial marker from their live
image stream. To keep the pose estimates as up-to-date and accurate as possible, four parameters are considered([](#performance_metrics)).
The goal is to reduce the delay and CPU usage, while increasing the frequency. It is also important that the quality of the
detector is still robust.
<figure id="performance_metrics">
    <figcaption>Fiducial Marker detection performance metrics</figcaption>
    <img alt="performance_metrics" style='width:32em' src="images/fiducial_markers_performance_metrics.png"/>
</figure>

## The fiducial markers of Duckietown {#fiducial-markers-what}
The fiducial markers that Duckietown uses are [AprilTags](https://april.eecs.umich.edu/software/apriltag). These tags are widely used in robotics, due to the fact that every tag as a unique identitiy, and each tag is made in a way to reduce the number of outliers detected. For more  information about the AprilTag specifications please read [this](https://docs.duckietown.org/daffy/downloads/opmanual_autolab/docs-opmanual_autolab/builds/211/opmanual_autolab/out/localization_apriltags_specs.html).


## How the AprilTag detector works {#apriltag-detector}
The AprilTag detector that Duckietown uses is [AprilTag3](https://april.eecs.umich.edu/media/pdfs/krogius2019iros.pdf) from AprilRobotics. This is a small c library with minimal dependencies. The GitHub repository can be found  [here](https://github.com/AprilRobotics/apriltag).

<figure id="at_detection">
    <figcaption>AprilTag3 detection step by step</figcaption>
    <img alt="apriltag3_detection" style='width:38em' src="images/apriltag3_steps.png"/>
</figure>


The way the detector works is that it takes a raw and rectified image ([](#at_detection): 1.input), then to increase the detection speed, it is possible to decimate the image, this means that it reduces the resolution of the image by a factor that the user can configure. Choosing  the  decimation  factor  allows  a  tradeoff between successful recall and speed. This happens in the preprocess step ([](#at_detection): 2.Preprocess).

Then it thresholds the image from a grayscale input image into a black-and-white image ([](#at_detection): 3.Threshold). For this application
it is only nessesary to seperate the light and dark pixels which form the tag. The areas of the image colored in grey are area with
insufficent contrast. These points are excluded from further processing to save time.

Connected components of same colored pixels (either black or white) are segmented together using  a  union-find  algorithm. Each connected component gets a unique ID ([](#at_detection): 4.Segmentation).

For every black and white connected component neighbours, the borderpixles form a cluster ([](#at_detection): 5.Clustering). The pixels that are not included in these clusters gets dropped. The clustering gets done efficiently by using a hash table. 

Now, for each cluster it tries to fit a quad ([](#at_detection): 6b.Quads). Since the AprilTag is a square, it tries to fit it inside the cluster, this
can be at an angle, so many different candidates gets picked. First the points are sorted by angle in a consistent winding order around their centroid. Corner points are identified by attempting to fit a line towindows of neighboring points. Line fits are computed using principal componentanalysis (PCA). The quad fitting step outputs a set of candidate quads for decoding

After the quad candidates a picked, it then uses the original image to sample if there is an
AprilTag inside the quad ([](#at_detection): 6c.Samples). 

Finally it outputs the detected tags in the image ([](#at_detection): 7.Output). It then uses the camera parameters to calculate the relative 
translation and rotation between the camera and the tag.

## Configurable parameters of the detector {#apriltag-params}
Apriltag3 has some configurable parameters that can increase the speed of the detector, but reduce the robustness. Follow [this link](https://github.com/duckietown/lib-dt-apriltags) to find out more about how each parameter affect the detector. These configurations are specific to every usecase, and therefore it needs to be tuned to fit the Autolab condtitions.

Extensive tests in different lighting conditions has been conducted, and the optimal configuration for the use of the detector in an Autolab is found to be:
<div figure-id="tab:apriltag3-param" markdown="1">
  <style>
    td:nth-child(2) {
    white-space: pre;
   }
  </style>
  <col3 class="labels-row1" >
    <span>Parameter name </span>
    <span>Value</span>
    <span>Explanation</span>
    <span>`nthreads`</span>
    <span>`3`</span>
    <span> Number of threads</span>
    <span>`quad_decimate`</span>
    <span>`2.0`</span>
    <span>Lowers the resolution of the image that is used to detect quads. Decoding the binary payload is still done at full resolution. A decimation of 2 will reduce the pixels to 1/4th of the original image</span>
    <span>`quad_sigma`</span>
    <span>`0.0`</span>
    <span>What Gaussian blur should be applied to the segmented image.</span>
    <span>`refine_edges`</span>
    <span>`1`</span>
    <span>When non-zero (1), the edges of the each quad are adjusted to "snap to" strong gradients nearby. </span>
    <span>`decode_sharpening`</span>
    <span>`0.25`</span>
    <span>How much sharpening should be done to decoded images.</span>
  </col3>
  <figcaption>Apriltag3 Parameters</figcaption>
</div>

To get a detailed overview of how each parameter affect the detection speed and performance, read assessment 2 (Tag detector parameters) [here](https://ethidsc.atlassian.net/wiki/spaces/DS/pages/465371137/Design+Document+The+on-board+Apriltag+Processing#Assessment-2%2C-Tag-detector-parameters%3A).

## Localization in Duckietown {#localization-in-duckietown}
The localization system of the Autolab are there to detect poses of all
Autobots in the town. This is done by cameras that are mounted on Watchtowers
and they are detecting fiducial markers in the form of AprilTag family 'tag36h11' both on the Autobots and the ground itself.
A more detailed overview of how the localization system works, as well as how to set
up an Autolab is found 
[here](https://docs.duckietown.org/daffy/downloads/opmanual_autolab/docs-opmanual_autolab/builds/211/opmanual_autolab/out/autolab_localization.html).

## The localization Pipeline {#localization-pipeline}
The high-level localization pipeline consists of three nodes, two on each watchtower, and one on the town/server like shown in [](#localization_pipeline).
The camera, which is being run by the camera node, captures images at 30 Hz at a resolution of (1296x972) which is the default [Watchtower configuration](https://docs.duckietown.org/daffy/downloads/opmanual_autolab/docs-opmanual_autolab/builds/211/opmanual_autolab/out/watchtower_initialization.html). The camera node compresses the image with JPEG compression, and sends the compressed image and the camera info containing the calibration information about the camera to the AprilTag detector node. The AprilTag detector node decompresses the image, and uses the camera info to detect the tags on the image. An [array of Apriltaginfo](https://github.com/duckietown/dt-ros-commons/blob/daffy-new-deal/packages/duckietown_msgs/msg/AprilTagDetectionArray.msg) of all the tags detected in that image then gets sent to the graph optimizer, which can be on a different computer, where it calcluates the tags relative pose and trajectory using the detected tags from all the Watchtowers in the town. 
<figure id="localization_pipeline">
    <figcaption>online Localization Pipeline</figcaption>
    <img alt="localization_pipeline" style='width:32em' src="images/localization_pipeline.png"/>
</figure>

## Modifications done to the detector node {#apriltag-duckietown-detector}
To enable the use of AprilTag3 detector in Duckietown, a [Duckietown-specific Python wrapper](https://github.com/duckietown/lib-dt-apriltags) has been made. This wrapper enables the use of the detector function within Python, and therefore directly from the detector node inside [dt-core](https://github.com/duckietown/dt-core) which is the core stack of the Duckiebot and Watchtowers. These changes enables the detector to detect tags in rectified images. 

With the pipeline described in [The localization Pipeline](#localization-pipeline), one need to modify Apriltag3 c-library to enable detection on non-rectified images. The way this is done is to create a mapping between the pixels in the non-rectified image and the pixels in the rectied image. This map can be found using the camera info that is sent from the camera node. The detector already uses the Camera Parameters (fx, fy, cx, cy) to enable the retreval of the relative pose estimates between the camera and the AprilTag. By modifying the wrapper to send the whole camera info (distorion coefficients and projection matrix) to the AprilTag3, one can now create a mapping inside the detector, which will map every incomming pixel to their relative rectified pixel coordinate. To use this new added feature, one needs to initialize the mapping by using the following function:

    Detector.enable_rectification_step(image_width, image_height, K, D, P)

This function is only needed to be called once. When this function is called, the AprilTag3 will assume that the incomming image is non-rectified and therefore will rectify the image using mappings created inside Apriltag3.

## Why Rectify inside the detector node {#apriltag-detector-rectification}

The benefits of rectifying the image inside of the detector, instead of feeding the Detector a prerectified image, is that the [designchoice](#apriltag-params) of Duckietown is to use a decimation of 2, which means that the total pixel number to rectify is reduced to 1/4th of the original image. This way, the rectification speed up, and the detector becomes faster and can process more frames per second. Another reason is that this reduces the complexity of the overall pipeline. By only sending the image directly from the camera node to the Detector, one needs less nodes running and hence more free CPU on the Raspberry Pi.



