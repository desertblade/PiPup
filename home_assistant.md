# AndroidTV/FireTV Notifications From Frigate

Here is how I setup FireTV notivications in Home Assistant automation to stream a camera feed from Frigate.


### Background
I first followed this [excellent write up from Sean](https://seanblanchfield.com/2022/03/realtime-pip-cameras-on-tv-with-home-assistant). The WebRTC stream was difficult to set up and was to slow to load on my FireTV (3 seconds).

I started looking into using [Frigate's MJPEG Streams](https://docs.frigate.video/integrations/api/#get-apicamera_name), which required modifing the APK to allow non SSL sites. I then expanded this code base to support Javascript for those who want to still use WebRTC and better handling of closing of the WebView.

# Install APK

Install [ADB](https://www.xda-developers.com/install-adb-windows-macos-linux/#adbsetup) on your computer

Enable Developer Options on your AndroidTV/[FireTV](https://www.xda-developers.com/how-to-access-developer-options-amazon-fire-tv/)

Download the APK from the [release page](https://github.com/desertblade/PiPup/releases)

In a Terminal run the following commands to install the APK:

``` terminal
./adb connect FIRETV_IP

./adb install app-debug.apk

./adb shell appops set nl.rogro82.pipup SYSTEM_ALERT_WINDOW allow
```

Windows users probably don't need the ./

Fire up the App on the FireTV. If you want to test it there are good instrcutions in the write up above.

# Home Assistant 

## Rest Commands
I am using the same ones in Sean's writeup. 

In the configuration.yaml add this line in the bottom (obvs if you don't already have it)

``` yaml
rest_command: !include rest_commands.yaml
```

Then add the following to rest_commands.yaml
``` yaml
pipup_image_on_tv:
  # Use Pipup to display image notifications on Android TV devices.
  url: http://ANDROID_TV_IP_ADDRESS:7979/notify
  content_type: 'application/json'
  verify_ssl: false
  method: 'post'
  timeout: 20
  payload: >
    {
      "duration": {{ duration | default(20) }},
      "position": {{ position | default(0) }},
      "title": "{{ title | default('') }}",
      "titleColor": "{{ titleColor | default('#50BFF2') }}",
      "titleSize": {{ titleSize | default(10) }},
      "message": "{{ message }}",
      "messageColor": "{{ messageColor | default('#fbf5f5') }}",
      "messageSize": {{ messageSize | default(14) }},
      "backgroundColor": "{{ backgroundColor | default('#0f0e0e') }}",
      "media": { 
        "image": {
          "uri": "{{ url }}",
          "width": {{ width | default(640) }}
        }
      }
    }


pipup_url_on_tv:
  # Use PipUp to display web pages/videos
  url: http://ANDROID_TV_IP_ADDRESS:7979/notify
  content_type: 'application/json'
  verify_ssl: false
  method: 'post'
  timeout: 20
  payload: >
    {
      "duration": {{ duration | default(20) }},
      "position": {{ position | default(0) }},
      "title": "{{ title | default('') }}",
      "titleColor": "{{ titleColor | default('#50BFF2') }}",
      "titleSize": {{ titleSize | default(10) }},
      "message": "{{ message }}",
      "messageColor": "{{ messageColor | default('#fbf5f5') }}",
      "messageSize": {{ messageSize | default(14) }},
      "backgroundColor": "{{ backgroundColor | default('#0f0e0e') }}",
      "media": { 
        "web": {
          "uri": "{{ url }}", 
          "width": {{ width | default(640) }},
          "height": {{ height | default(480) }}
        }
      }
    }
```

## Automation

Using MJPEG streams from Frigate I found the fastest to load.

Here is the automation I use. Change media_player.firetv_cube to your device and update the Frigate URL IPs and Cameras.

``` yaml
- alias: Doorbell Person Alerts
  trigger:
    platform: mqtt
    topic: frigate/events
  condition:
    # Only if the FireTV is On
    - "{{states('media_player.firetv_cube') != 'off'}}"
    # To reduce duplicate alerts, only every 60 seconds
    - '{{ (as_timestamp(now()) - as_timestamp(states.automation.doorbell_person_alerts.attributes.last_triggered, 0 ) | int > 60)}}'
    #Only show for a person and on doorbell camera
    - "{{ trigger.payload_json['after']['label'] == 'person' }}"
    - "{{ trigger.payload_json['after']['camera'] == 'doorbell' }}"
  action:
  # Make sure PipUp is running
    - service: androidtv.adb_command                         
      data:                            
        entity_id: media_player.firetv_cube                   
        command: ps -ef | grep -v grep | grep pipup || am start nl.rogro82.pipup/.MainActivity
  # This will exit the screensaver
    - service: androidtv.adb_command
      data:
        entity_id: media_player.firetv_cube
        command: input keyevent KEYCODE_WAKEUP
    - service: rest_command.pipup_url_on_tv
      data:
        title: Front Door
        message: Live Stream of Doorbell
        width: 360
        height: 200
        url: 'http://192.168.X.XX:5000/api/doorbell'
```

Once Automation is saved and loaded you can manually trigger it from the Admin page in Home Assistant to test.