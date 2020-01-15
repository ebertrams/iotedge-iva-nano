# Visual Anomaly Detection over multiple cameras with NVIDIA Jetson Nano devices workshop

In this workshop, you'll discover how to build a solution that can process up to 8 real-time video streams with an AI model on a $100 device, how to remotely operate your device, and demonstrate how you can deploy custom AI models to it. 

With this solution, you can transform cameras into sensors to know when there is an available parking spot, a missing product on a retail store shelf, an anomaly on a solar panel, a worker approaching a hazardous zone., etc.

We'll build this solution using [NVIDIA Deepstream](https://developer.nvidia.com/deepstream-sdk) on a [NVIDIA Jetson Nano device](https://developer.nvidia.com/embedded/buy/jetson-nano-devkit) connected to Azure via [Azure IoT Edge](https://azure.microsoft.com/en-us/services/iot-edge/). Deepstream is an highly-optimized video processing pipeline, capable of running deep neural networks. It is a must-have tool whenever you have complex video analytics requirements like real-time object detection or when employing cascading AI models. IoT Edge gives you the possibility to run this pipeline next to your cameras, where the video data is being generated, thus lowering your bandwitch costs and enabling scenarios with poor internet connectivity or privacy concerns.

We'll operate this solution with an aesthetic UI provided by [IoT Central](https://azure.microsoft.com/en-us/services/iot-central/) and customize the objects detected in video streams using [Custom Vision](https://www.customvision.ai/), a service that automatically generates computer vision AI models from pictures.

## Prerequisites

![Jetson Nano](./assets/JetsonNano.png "NVIDIA Jetson Nano device used to run Deepstream with IoT Edge")

- **Hardware**: You need a [NVIDIA Jetson Nano device](https://developer.nvidia.com/embedded/buy/jetson-nano-devkit) with a [5V-4A barrel jack power supply like this one](https://www.adafruit.com/product/1466), which are provided in this workshop.
- **A laptop**: You need a laptop to connect to your Jetson Nano device and see its results with a browser and an ssh client.
- **VLC**: To visualize the output of the Jetson Nano without HDMI screen (there is only one per table), we'll use VLC from your laptop to view a RTSP video stream of the processed videos. [Install VLC](https://www.videolan.org/vlc/index.html) if you dont have it yet.
- **Access to IoT Central**: Jetson Nano devices have been prepared to connect to the following IoT Central application: [https://deepstream-on-iot-edge.azureiotcentral.com/](https://deepstream-on-iot-edge.azureiotcentral.com/). Please ask for help if you cannot login to this application with your Microsoft account.

Note: Your NVIDIA Jetson Nano has already been flashed with the latest image (Jetpack 4.3). It is based on Ubuntu 18.04 and already includes NVIDIA drivers, CUDA, and Nvidia-Docker. For this workshop, we've also added several other files to speed things up.

## Running the solution

In the interest of time, we've already built a first solution that is composed of two main blocks:

1. **NVIDIA DeepStream** to do all the video processing

    The first component, DeepStream, was trivial to build since it has been published by NVIDIA in the [Azure Marketplace here](https://azuremarketplace.microsoft.com/en-us/marketplace/apps/nvidia.deepstream-iot?tab=Overview). We're using this module as-is and are only configuring it from the IoT Central bridge module.

![Deepsteam in Azure Marketplace](./assets/DeepstreamInMarketplace.png "NVIDIA Deepstream in Azure Marketplace")

1. A **bridge to IoT Central** to transform DeepStream telemetry into a format understood by IoT Central and to configure DeepStream remotely. It formats all telemetry, properties, and commands using PnP. 

Let's see it in action!

1. Plug your device, press the power button and give it a few minutes to boot. It should automatically connect to the conference's wifi, start its application thanks to IoT Edge and report back its usage and IP address to IoT Central.
2. Connect to [this IoT Central application](https://deepstream-on-iot-edge.azureiotcentral.com/) with your Microsoft account. Please ask for help if you dont have access.
3. Go to `Devices` and find the device number corresponding to your Jetson Nano (You should have received this number from the proctors and will range from 01-45, for example if you received 01 then your device will be jetson-nano-01)
4. Click on this device and go to the `Dashboard` tab
5. Verify that active telemetry is being sent by the device to IoT Central
6. Copy the `RTSP Video URL` from the `Device` tab
7. Open VLC and go to `Media` > `Open Network Stream` and paste the `RTSP Video URL` copied above as the network URL and click `Play`

At this point, you should see 8 video streams being processed to detect cars and people with a Resnet 10 AI model.

![8 video streams processed real-time](./assets/8VideosProcessed.png "8 video streams processed in real-time by a Jetson Nano with Deepstream and IoT Edge")

### Understanding NVIDIA DeepStream

Deesptream is a SDK based on GStreamer. It is very modular with its concepts of plugins. Each plugins having `sinks` and `sources`. NVIDIA provides several plugins as part of Deepstream which optimized to leverage NVIDIA's GPUs. How these plugins are connected with each others in defined in the application's configuration file.

You can learn more about its architecture in [NVIDIA's official documentation](https://docs.nvidia.com/metropolis/deepstream/dev-guide/index.html#page/DeepStream_Development_Guide%2Fdeepstream_app_architecture.html) (sneak peak below).

![NVIDIA Deepstream Application Architecture](./assets/DeepStreamArchitecture.png).

To better understand how NVIDIA DeepStream works, let's have a look at its configuration file.

From your favourite SSH client:

1. Open an SSH connection with the IP address found in the `RTSP Video URL` field above. Username is `iotedge` and so is the default password.

    ```bash
    ssh iotedge@YOUR_IP_ADDRESS
    ```

2. Open up the configuration file of DeepStream to understand its structure:
    
    ```bash
    nano /data/misc/storage/DSconfig.txt
    ```

Observe in particular:

   - the `sources` sections: they define where the source videos are coming from. We're using local videos to begin with and will switch to live RTSP streams later on.
   - the `sink` sections: they define where to output of the processed videos and the output messages. We use RTSP to stream a video feed out and all out messages are sent to the Azure IoT Edge runtime.
   - the `primary-gie` section: it defines which AI model is used to detect objects. It also defines how this AI model is applied. As an example, note the `interval` property set to `4`: this means that inferencing is actually executed only once every 5 frames. Bounding boxes are displayed continuously though because a tracking algorithm, which is computationally less expensive than inferencing, takes over in between. The tracking algorithm used is set in the `tracking` section. This is the kind of out-of-the-box optimizations provided by DeepStream that enables us to process 240 frames per second on a $100 device. Other notable optimizations are using hardware decoders, doing zero in-memory copy, pushing the vast majority of the processing to the GPU, batching frames from multiple streams, etc.

### Understanding the connection to IoT Central

IoT Edge connects to IoT Central with the regular Module SDK. Telemetry, Properties and Commands that the IoT Edge Central bridge module sends follow a PnP format, enforced in the Cloud by IoT Central. IoT Central enforces them against a Device Capability Model, which is a file that defines what this IoT Edge device is capable of doing.

- Click on `Devices` in the left nav of the IoT Central application
- Observe the templates in the second column: they define all the devices that this IoT Central application understands. All the Jetson Nano devices of this workshop are using a version of the `NVIDIA Jetson Nano (Airlift)` device template. In the case of IoT Edge, an IoT Edge deployment manifest is attached to a DCM version to create a device template. If you want to see the details on how a Device Capability Model look like, you can look at (this file in this repo)[https://github.com/ebertrams/iotedge-iva-nano/blob/IoTCentralBridge-CV/settings/NVIDIAJetsonNanoDcm.json]. 

## Operating the solution

Now that we understand how the application has been built, let's operate it with IoT Central. To demonstrate how to remotely manage this solution, we'll send a command to the device to change its input cameras.

![IoT Central](./assets/IoTCentral.png "IoT Central UI to remotely manage NVIDIA Jetson devices")

### Changing input cameras

Before switching the application to a different camera, let's just verify that the camera is functional. With VLC: 

- Go to `Media` > `Open Network Stream`
- Paste the following `RTSP Video URL`:  rtmp://visionaistreams.westus2.cloudapp.azure.com/visionai/west-team-room-srt-2600.stream
- Click `Play` and verify that an indoor camera is properly displaying. If this video feed does not work, please move on to the next section.

You can do the same with the second indoor camera: rtmp://visionaistreams.westus2.cloudapp.azure.com/visionai/east-team-room-srt-2601.stream

Now, let's update the solution to use these indoor cameras. In IoT Central:

- Go to the `Manage` tab
- Unselect the `Demo Mode`, which uses 8 hardcoded video files as input of car traffic
- Update the `Video Stream 1` property:
    - In the `cameraId`, name your camera, for instance `indoor Camera 1`
    - In the `videoStreamUrl`, enter the RTSP stream of this camera: rtmp://visionaistreams.westus2.cloudapp.azure.com/visionai/west-team-room-srt-2600.stream
- Update the `Video Stream 2` property:
    - In the `cameraId`, name your camera, for instance `indoor Camera 2`
    - In the `videoStreamUrl`, enter the RTSP stream of this camera: rtmp://visionaistreams.westus2.cloudapp.azure.com/visionai/east-team-room-srt-2601.stream
- Keep the default AI model of DeepStream by keeping the value `DeepStream ResNet 10` as the `AI model type`.
- Keep the default `Primary Detection Class` as `person`
- Hit `Save`

This sends a command to the device to update its DeepStream configuration file with these new properties and to restart DeepStream. If you were still streaming the output of the DeepStream application, this streem will be taken down as DeepStream will restart.

Within a minute, DeepStream should restart. You can observe its status in IoT Central via the `Modules` tab. Once `deepstream` module is back to `Running`, copy again the `RTSP Video Url` field from the `Device` tab and give it to VLC (`Media` > `Open Network Stream` > paste the `RTSP Video URL` > `Play`).

You should now detect people from 2 indoor cameras. The count of `Person` in the `dashboard` tab in IoT Central should go up. We've just remotely updated the configuration of this intelligent video analytics solution!

## Use an AI model to detect custom visual anomalies

A soda can manufaturer wants to improve the efficienty of its plant. He would like to be able to take soda cans that fell down on his production lines to avoid slow downs.
We'll use cameras to monitor each of the lines and we'll collect images and build a custom AI model to detects cans that are up or down. We'll then deploy this custom AI model to DeepStream via IoT Central. To do a quick Proof Of Concept, we'll use the [Custom Vision service](https://www.customvision.ai/), a no-code computer vision AI model builder.

As a pre-requisite, let's create a new Custom Vision project in your subscription:
- Go to [http://customvision.ai](https://www.customvision.ai/)
- Sign-in
- Create a new Project
- Give it a name like `Soda Cans Down`
- Pick up your resource, if none select `create new` and select `SKU - F0` or (S0)
- Select `Project Type` = `Object Detection`
- Select `Domains` = `General (Compact)`

We then need to collect images to build a custom AI model. To do that, we'll use a utility that captures images from RTSP video streams. It has already been deployed on the Jetson Nano as a module named `CameraTaggingModule`. To acces it, go to the `Device` tab in IoT Central and copy/paste the `Video Tagging Client Url` into your laptop's browser.

From the Camera Tagging Module:

- Connect to one of the camera of the manufacturing line:
   - 'Change camera' (you may have to scroll down to see the button)
   - Add a new camera with the following RTSP value: rtsp://cam-cans-01.westus.azurecontainer.io:8554/live.sdp
- Capture a few images of cans that are up and cans that are down
- Once a few images have been capture, go to upload and enter your Custom Vision settings
- Finally, go to the image gallery, select the ones that you want to upload and upload them to Custom Vision

In the interest of time, [here](https://1drv.ms/u/s!AEzLzaBpSgoMo-1l) (https://1drv.ms/u/s!AEzLzaBpSgoMo-1l) is a set of images that have already been captured for you that you can upload to Custom Vision. Download it, unzip it and upload all the images into your Custom Vision project.

We then need to label all of them:
- Click on one image
- Label the cans that are up as `Up` and the ones that are down as `Down`
- Hit the right arrow to move on to the next image and label the remaining 70+ images...or read below to use a pre-built one with this set of images
- Once you're done labeling, click on `Train`
- To export it, go to the `Performance` tab, click on `Export` and choose `ONNX`
- Right-click on the `Download` button and select `copy link address` to copy the anonymous location of a zip file of your ccustom model

In the interest of time, you can use the following pre-built Custom Vision model for this use case:
https://irisprodwu2training.blob.core.windows.net/m-cab1e73a2d5043709139ca73f8fdcd1a/d26d3cdca28a4c3d815393cbfc853f1c.ONNX.zip?sv=2017-04-17&sr=b&sig=hVE6YzZBVsB2WciZExHZWeWvY1UjZUQB4MdbtooD%2Fj8%3D&se=2020-01-15T15%3A46%3A48Z&sp=r

Finally, we'll deploy this custom vision model to the Jetson Nano using IoT Central. In IoT Central:
- Go to the `Manage` tab (beware of the sorting o f the fields)
- Make sure the `Demo Mode` is unchecked
- The 3 input video for 3 manufacturing lines have been hardcoded into the device to avoid network issues. You would normally have to input valid RTSP input sources in the first 3 Video Stream Url properties. You can put any values for now in these fields.
- Select `Custom Vision` as the `AI model Type`
- Paste the URI of your custom vision model in the `Custom Vision Model Url`
- Update the `Primary Detection Class` to be `Up` and the `Secondary Detection Class` to be `Down`
- Hit `Save`

After a few moments, the `deepstream` module should restart. Once it is in `Running` state again, look at the output RTSP stream via VLC (`Media` > `Open Network Stream` > paste the `RTSP Video URL` that you got from the IoT Central's `Device` tab > `Play`).

We are now visualizing the processing of 3 real time (e.g. 30fps 1080p) video feeds with a custom vision AI models that we built in minutes to detect visual anomalies!

![Custom Vision](./assets/sodaCans.png "3 soda cans manufacturing lines are bieing monitored with a custom AI model built with Custom Vision")


Thank you for attending this workshop! There are other content that you can try with your Jetson Nano at [http://aka.ms/jetson-on-azure](http://aka.ms/jetson-on-azure)!

# APPENDIX A - Setup your Jetson Nano device

The Jetson Nano devices have been flashed with almost everything that is needed. A few extra steps are required though to finalize its configuration:

- Update its hostname. Replace `00` in instructions below by your device number:

```bash
sudo hostnamectl set-hostname jetson-nano-00
sudo nano /etc/hosts
```
    
    - Edit the hostname in /etc/hosts
    - Save (CTRL+O) and exit (CTRL+X).

- Edit the permissions of the IoT Edge configuration file:

```bash
sudo chmod 755 /etc/iotedge/config.yaml
```

- Edit the IoT Edge configuration file:

```bash
sudo nano /etc/iotedge/config.yaml
```

    - Update the registration_id in the provisioning section to be your device hostname `jetson-nano-00`
    - Copy the device key from IoT Central (Go to the device > click `Connect` in the top right corner and copy the `Primary key`) and paste it as the symmetric_key in the provisioning section
    - Towards the bottom of the configuration file, update the device hostname


- Reboot

Your device should now be connected to IoT Central. Go to its dashboard to verify that it is the case (it may take a couple of minutes).

