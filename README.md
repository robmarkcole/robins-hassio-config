# robins-hassio-config
My home-assistant config from my experimental hassio instance.

My [primary](https://github.com/robmarkcole/robins-homeassistant-config) home-assistant instance is running on a synology NAS. I run a Hassio instance on a pi 3 for experiments and testing of new integrations.

## Motion detection with a USB camera
One of the main reasons I setup the Hassio instance was to build a usb camera based motion detection and alert system. I have a cheap ([10 pounds on Amazon](https://www.amazon.co.uk/gp/product/B000Q3VECE/ref=oh_aui_detailpage_o02_s00?ie=UTF8&psc=1)) usb webcam that captures images on motion detection [using](https://community.home-assistant.io/t/usb-webcam-on-hassio/37297/7) the [motion](https://motion-project.github.io/) hassio addon.

##### Motion addon setup
I've configured the [motion](https://github.com/HerrHofrat/hassio-addons/tree/master/motion). add-on (via its hassio tab) with the following settings:

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
This setup captures an image every second, saved as latest.jpg and over-written every second. Additionally, on motion detection a time stamped image is saved for format ```%v-%Y_%m_%d_%H_%M_%S-motion-capture```.

##### HA config

The image ```latest.jpg``` (updated and over-written every second) is displayed on the HA front-end using a [local-file camera](https://home-assistant.io/components/camera.local_file/). I also display the last motion captured image with a second file_sensor camera. **Note** that the image file (i.e. ```latest.jpg and MOTION.jpg```) must be present when HA starts as the component makes a check that the file exists. TO DO - ADD SCRIPT TO COPY THE LATEST MOTION TRIGGERED IMAGE FILE AND OVERWRITE MOTION.jpg

```yaml
camera:
  - platform: local_file
    file_path: /share/motion/latest.jpg
    name: "Live view"
  - platform: local_file
    file_path: /share/motion/MOTION.jpg
    name: "Last captured motion"
```
I then use my [folder sensor custom component](https://github.com/robmarkcole/HASS-folder-sensor) to detect when new motion triggered images are saved:

```yaml
sensor:
  - platform: folder
    folder: /share/motion
    filter: '*capture.jpg'
```

I then add a [counter](https://home-assistant.io/components/counter/) to count the number of new motion captured images.

```yaml
counter:
  motion_counter:
    initial: 0
    step: 1
```
I used the automations editor to create an automation which increments the counter every time there is a state change of my folder sensor. I use a second automation to reset the counter every day.

I then add a [template sensor](https://home-assistant.io/components/sensor.template/) to display the total number of motion images in the folder - if this gets very large I will delete some images to save disk space. I also add a template sensor to display the full path to the last captured image, as I will use this to display the image shortly.

```yaml
sensor:
  - platform: template
    sensors:
      number_of_files_motion:
        friendly_name: "Motion images"
        value_template: "{{states.sensor.motion.attributes.number_of_files}}"
      last_captured_image:
        friendly_name: "Last captured image"
        value_template: "{{states.sensor.motion.attributes.folder + states.sensor.motion.attributes.modified_file}}"
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
