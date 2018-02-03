# robins-hassio-config
My home-assistant config from my experimental hassio instance.

My [primary](https://github.com/robmarkcole/robins-homeassistant-config) home-assistant instance is running on a synology NAS, as this is a very stable platform. I run a Hassio instance on a pi 3 for experiments and testing.

## Motion detection with a USB camera
One of the main reasons I setup the Hassio instance was to build a usb camera based motion detection and alert system. I have a cheap ([10 pounds on Amazon](https://www.amazon.co.uk/gp/product/B000Q3VECE/ref=oh_aui_detailpage_o02_s00?ie=UTF8&psc=1)) usb webcam that captures images on motion detection [using](https://community.home-assistant.io/t/usb-webcam-on-hassio/37297/7) the [motion](https://motion-project.github.io/) hassio addon.

##### Motion addon
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
  "picture_name": "%v-%Y%m%d%H%M%S-motion-capture",
  "webcontrol_local": "on",
  "webcontrol_html": "on"
}
```
This config captures an image every second, saved as latest.jpg and over-written every second. Additionally well on motion detection a time stamped image is saved for format "%v-%Y%m%d%H%M%S-motion-capture.jpg".

The image latest.jpg (updated and over-written every second) is displayed on the HA front-end using a [local-file camera](https://home-assistant.io/components/camera.local_file/).

```yaml
camera:
  - platform: local_file
    file_path: /share/motion/latest.jpg
```
I then use my [folder sensor custom component](https://github.com/robmarkcole/HASS-folder-sensor) to detect when new motion triggered images are saved:

```yaml
sensor:
  - platform: folder
    folder: /share/motion
    filter: '*capture.jpg'
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
