# robins-hassio-config
My home-assistant config from my experimental hassio instance. Validated at HA 0.67.

My [primary](https://github.com/robmarkcole/robins-homeassistant-config) home-assistant instance is running on a synology NAS. I run a Hassio instance on a pi 3 for experiments and testing of new integrations.

# Image classification
I'm using a [custom component](https://github.com/robmarkcole/HASS-Machinebox-Classificationbox) to perform image classification on images captured by my motion triggered camera. I've trained a [Classificatiobox](https://machinebox.io/docs/classificationbox) classifier with images captured using the motion setup below. The classifier can detect whether or not there is a bird in a captured image. The classifier has an accuracy of about 90%, trained on 1000 images that I manually sorted.

## Motion detection with a USB camera
One of the main reasons I setup the Hassio instance was to build a usb camera based motion detection and alert system. I have a cheap ([10 pounds on Amazon](https://www.amazon.co.uk/gp/product/B000Q3VECE/ref=oh_aui_detailpage_o02_s00?ie=UTF8&psc=1)) usb webcam that captures images on motion detection [using](https://community.home-assistant.io/t/usb-webcam-on-hassio/37297/7) the [motion](https://motion-project.github.io/) hassio addon. The final view in HA is shown below, with the live view camera image just cropped off the bottom of the image.

<p align="center">
<img src="https://github.com/robmarkcole/robins-hassio-config/blob/master/images/HA_motion_camera_view.png" width="500">
</p>

##### Motion addon setup
I've configured the [motion](https://github.com/HerrHofrat/hassio-addons/tree/master/motion) add-on (via its hassio tab) with the following settings:

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
The image `latest.jpg` (updated and over-written every second) is displayed on the HA front-end using a [local-file camera](https://home-assistant.io/components/camera.local_file/). I also display the last motion captured image with a second `local_file` camera. **Note** that the image files (here `latest.jpg` and `MOTION.jpg`) must be present when HA starts as the component makes a check that the file exists, and therefore if running for the first time just copy some appropriately named images into the `/share/motion` folder. In `configuration.yaml`:

```yaml
camera:
  - platform: local_file
    file_path: /share/motion/latest.jpg
    name: "Live view"
  - platform: local_file
    file_path: /share/motion/MOTION.jpg
    name: "Last captured motion"
```
I use the [folder_watcher component](https://www.home-assistant.io/components/folder_watcher/) to detect when new motion triggered images are saved:

```yaml
folder_watcher:
  - folder: /share/motion
    patterns:
      - '*capture.jpg'
```

The `folder_watcher` fires an event with the event data including the image path to the added file. I use an automation to display the new image on the `local_file` camera using the `camera.update_file_path` service:

```yaml
- action:
    data_template:
      file_path: ' {{ trigger.event.data.path }} '
    entity_id: camera.local_file
    service: camera.update_file_path
  alias: Display new image
  condition: []
  id: '1520092824633'
  trigger:
  - event_data:
      event_type: created
    event_type: folder_watcher
    platform: event
```

I use a template sensor to break out the new file path:
```yaml
sensor:
  - platform: template
    sensors:
      last_added_file:
        friendly_name: Last added file
        value_template: "{{states.camera.local_file.attributes.file_path}}"
```

I then use an automation triggered by the state change of the tempplate sensor to send me the new image as a Pushbullet notification:

```yaml
- action:
  - data_template:
      message: New image {{ states.camera.local_file.attributes.file_path }}
      title: New image on camera.local_file
      data:
        file: ' {{ states.camera.local_file.attributes.file_path }} '
    service: notify.pushbullet
  alias: Updated camera.local_file
  condition: []
  id: '1524081104601'
  trigger:
  - entity_id: sensor.last_added_file
    platform: state
```

The final view in HA is that shown at the top of this section. A photo of the setup is shown below.

<p align="center">
<img src="https://github.com/robmarkcole/robins-hassio-config/blob/master/images/camera_setup.jpg" width="500">
</p>

## HomeBridge
I use the [HomeBridge](https://github.com/hassio-addons/addon-homebridge) addon to integrate my HomeKit devices into HA. In particular I own the [Elgato door sensor](https://www.elgato.com/en/eve/eve-door-window) and the [Fibaro flood sensor](https://homekit.fibaro.com/). Both are binary sensors and the process of integrating them with HA, so I will here just discuss the door sensor.

##### Setup HomeBridge
On following the docs, I encountered a problem (discussed [in this forum post](https://community.home-assistant.io/t/community-hass-io-add-on-homebridge/33803/111)) where I kept receiving an error in the HomeBridge logs ```Not a valid username```. The problem was fixed by entering a valid MAC address (I created one randomly, also create a new one if you have to reinstall HomeBridge) in ```config.json```. Then start the addon, and add home-assistant as a bridge accessory in the iOS Home app by selecting ```Add Accessory```, then ```Manual Code```, and entering the 8 digit code shown in the HomeBridge logs. I created a room called HASS to put all my home-assistant devices in.

##### Home-assistant Usage
I use HomeBridge to toggle input_booleans in home-assistant, allowing HA to read the state of my HomeKit binary sensors. The method is described in [this forum thread](https://community.home-assistant.io/t/triggar-ha-from-homekit-devices/3253/11). I first configure (in home-assistant) an [input_boolean](https://home-assistant.io/components/input_boolean/) switch for each HomeKit device:

```yaml
input_boolean:
  elgato_door:
    name: Elgato door sensor
    icon: mdi:door
```

Now in the iOS Home app create a HomeKit automation to toggle your HA input_boolean when the HomeKit device (here the Elgato door sensor) has a state change. The automation for the door opening is shown below. Add another automation for the door closing.

<p align="center">
<img src="https://github.com/robmarkcole/robins-hassio-config/blob/master/images/HomeKit_add_door.jpg" width="300">
</p>

Now when your HomeKit device changes state, this will be mirrored in the home-assistant input_boolean. I use a [template binary sensor](https://home-assistant.io/components/binary_sensor.template/) to display the actual door state on the front-end:

```yaml
binary_sensor:
  - platform: template
    sensors:
      front_door:
        friendly_name: "Front door"
        value_template: "{{is_state('input_boolean.elgato_door', 'on')}}"
```

As my main HA install is a second HA instance (on a synology NAS), I add the front door to my main HA using the HA rest API, using a [restful binary sensor](https://home-assistant.io/components/binary_sensor.rest/):
```yaml
binary_sensor:
  - platform: rest
    scan_interval: 1
    resource: http://HA_IP:8123/api/states/binary_sensor.front_door
    name: front_door
    value_template: '{{value_json.state}}'
```

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
If you edit the config and restart HA but don't see desired changes, then reboot the pi with ```hassio homeassistant reboot```.
