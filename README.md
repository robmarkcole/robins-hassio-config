# robins-hassio-config
My home-assistant config from my experimental hassio instance. Validated at HA 0.92.

My [primary](https://github.com/robmarkcole/robins-homeassistant-config) home-assistant instance is running on a synology NAS. I run a Hassio instance on a pi 3 for experiments and testing of new integrations.

# Image classification
I'm using a [custom component](https://github.com/robmarkcole/HASS-Machinebox-Classificationbox) to perform image classification on images captured by my motion triggered camera. I've trained a [Classificatiobox](https://machinebox.io/docs/classificationbox) classifier with images captured using the motion setup below. The classifier can detect whether or not there is a bird in a captured image. The classifier has an accuracy of about 90%, trained on 1000 images that I manually sorted. My classifier component is configured with:

```yaml
image_processing:
  - platform: classificationbox
    ip_address: 192.168.0.18
    port: 8080
    scan_interval: 100000
    source:
      - entity_id: camera.dummy
```
With the long `scan_interval` I am ensuring that processing will only be performed when I trigger it with an automation.

## Motion detection with a USB camera
One of the main reasons I setup the Hassio instance was to build a usb camera based motion detection and alert system. I have a cheap ([10 pounds on Amazon](https://www.amazon.co.uk/gp/product/B000Q3VECE/ref=oh_aui_detailpage_o02_s00?ie=UTF8&psc=1)) usb webcam that captures images on motion detection [using](https://community.home-assistant.io/t/usb-webcam-on-hassio/37297/7) the [motion](https://motion-project.github.io/) hassio addon. The final view in HA is shown below, with the live view camera image just cropped off the bottom of the image.

<p align="center">
<img src="https://github.com/robmarkcole/robins-hassio-config/blob/master/images/HA_motion_camera_view.png" width="500">
</p>

##### Motion addon setup
I've configured the [motion](https://github.com/HerrHofrat/hassio-addons/tree/master/motion) add-on (via its Hassio tab) with the following settings:

```yaml
{
  "config": "",
  "videodevice": "/dev/video0",
  "input": 0,
  "width": 640,
  "height": 480,
  "framerate": 2,
  "text_right": "%Y-%m-%d %T-%q",
  "target_dir": "/share/motion",
  "snapshot_interval": 1,
  "snapshot_name": "latest",
  "picture_output": "best",
  "picture_name": "%v-%Y_%m_%d_%H_%M_%S-motion-capture",
  "webcontrol_local": "on",
  "webcontrol_html": "on"
}
```
This setup captures an image every second, saved as `latest.jpg`, and is over-written every second. Additionally, on motion detection a time-stamped image is captured with format `%v-%Y_%m_%d_%H_%M_%S-motion-capture.jpg`.

##### Home-Assistant config
The image `latest.jpg` is displayed on the HA front-end using a [local-file camera](https://home-assistant.io/components/camera.local_file/). I will also display the last time-stamped image with a second `local_file` camera. **Note** that the image files (here `latest.jpg` and `dummy.jpg`) must be present when HA starts as the component makes a check that the file exists, and therefore if running for the first time just copy some appropriately named images into the `/share/motion` folder. In `configuration.yaml`:

```yaml
camera:
  - platform: local_file
    file_path: /share/motion/latest.jpg
    name: "Live view"
  - platform: local_file
    file_path: /share/motion/dummy.jpg
    name: "dummy"
```
I use the [folder_watcher component](https://www.home-assistant.io/components/folder_watcher/) to detect when new time-stamped images are saved:

```yaml
folder_watcher:
  - folder: /share/motion
    patterns:
      - '*capture.jpg'
```

The `folder_watcher` fires an event with data including the image path to the added file. I use an automation to display the new image on the `dummy` camera using the `camera.update_file_path` service:

```yaml
- action:
    data_template:
      file_path: ' {{ trigger.event.data.path }} '
    entity_id: camera.dummy
    service: camera.local_file_update_file_path
  alias: Display new image
  condition: []
  id: '1520092824633'
  trigger:
  - event_data:
      event_type: created
    event_type: folder_watcher
    platform: event
```

I use a template sensor (in `sensors.yaml`) to break out the new file path:
```yaml
- platform: template
  sensors:
    last_added_file:
      friendly_name: Last added file
      value_template: "{{states.camera.dummy.attributes.file_path}}"
```
I use an automation triggered by the state change of the template sensor to trigger
image processing on the new image:

```yaml
- id: '1527837198169'
  alias: Perform image classification
  trigger:
  - entity_id: sensor.last_added_file
    platform: state
  condition: []
  action:
  - data:
      entity_id: camera.dummy
    service: image_processing.scan
```

Finally I use the event fired by the image classification to trigger an automation to send me the new image and classification as a Pushbullet notification:

```yaml
- action:
  - data_template:
      message: Class {{ trigger.event.data.class_id }} with probability {{ trigger.event.data.score }}
      title: New image classified
      data:
        file: ' {{states.camera.dummy.attributes.file_path}} '
    service: notify.pushbullet
  alias: Send classification
  condition: []
  id: '1120092824611'
  trigger:
  - event_data:
      event_type: image_classification
    event_type: image_processing
    platform: event
```

<p align="center">
<img src="https://github.com/robmarkcole/robins-hassio-config/blob/master/images/iphone_notification.jpeg" width="300">
</p>

A photo of my birdfeeder setup is shown below.

<p align="center">
<img src="https://github.com/robmarkcole/robins-hassio-config/blob/master/images/camera_setup.jpg" width="500">
</p>

## Hassio Add-ons
Try to limit your addons, as too many will make your system unstable.
1. IDE: https://github.com/hassio-addons/addon-ide
2. SSH & Web Terminal:  https://github.com/hassio-addons/addon-ssh/blob/v3.2.0/README.md
3. MQTT Server & Web client: https://github.com/hassio-addons/addon-mqtt/blob/master/README.md
4. Motion camera detection: https://github.com/HerrHofrat/hassio-addons
5. Glances: https://github.com/hassio-addons/addon-glances
6. Jupyterlab Lite: https://github.com/hassio-addons/addon-jupyterlab-lite
7. Log Viewer: https://github.com/hassio-addons/addon-log-viewer


## Tips
##### Recommended addons
At a minimum, I install the samba-share, ssh and configurator addons. I am also using the [motion](https://github.com/HerrHofrat/hassio-addons/tree/master/motion) addon.

##### Hassio disappears
If the Hassio tab disappears from the front end, add to your config:
```yaml
hassio:
```
and restart.

##### Hassio config changes dont take effect
If you edit the config and restart HA but don't see desired changes, then reboot the pi with ```hassio host reboot```
