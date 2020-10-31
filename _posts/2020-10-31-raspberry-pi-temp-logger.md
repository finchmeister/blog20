---
layout: post
title: Alexa Raspberry Pi Temperature Sensor
description: How to set up a Raspberry Pi Temperature Sensor
summary: How to set up a Raspberry Pi Temperature Sensor
tags: [tech, python, raspberrypi]
---

In this article I will share how I put together an Alexa connected Raspberry Pi temperature sensor using a novel data store.

<video width="320" controls>
    <source src="{{ '/assets/rpi-temp-logger/alexa-room-temp.mp4' | relative_url }}" type="video/mp4">
</video>

> "Alexa, room temperature?"
>
> **"The temperature is 18 degrees celcius (recorded 1 minute ago)"**

### Setting Up the Pi

For this project, I used an old Raspberry Pi 2 model with a USB WiFi dongle and a DHT22 temperature sensor.

The temperature sensor is connected to the Pi as follows:

```
Pi              DHT22 
---             ---      
Pin 1           3.3v power (+)
Pin 7 (GPIO4)   GPIO data out
Pin 9           Ground (-)
```

![Full commit]({{ '/assets/rpi-temp-logger/rpi-temp-sensor.jpg' | relative_url }})


I did the usual prep work to get remote SSH access to the Pi via Wifi

- Use Raspberry Pi Imager to prepare the SD card with Raspbian Lite
- Add `ssh` file to the root
- Create a `wpa_supplicant.conf` file with the wifi information

### Recording the Temperature

To access the temperature data, I used the [Python Adafruit DHT library](https://github.com/adafruit/Adafruit_Python_DHT) which I installed with:

```
git clone https://github.com/adafruit/Adafruit_Python_DHT.git && \
	cd Adafruit_Python_DHT && \
	python3 setup.py install && \
	cd ..
```

In its simplest form, here is how we can get the temperature from the sensor:

```python
import Adafruit_DHT

sensor = Adafruit_DHT.DHT22
sensor_gpio = 4

humidity, temperature = Adafruit_DHT.read_retry(sensor, sensor_gpio)
print("Temp: %.2f, Humidity: %.2f" % (temperature, humidity))

# Temp: 19.92, Humidity: 0.83
```

### Making the Data Accessible to Alexa

Unsurprisingly, Alexa needs access to the temperature data for her to voice it back. An option would be to expose the Pi to the web for Alexa to access the data directly but that involves a lot of faff setting up a web-server and messing around with router configuration to publicly expose said web-server.

Instead, I realised that a public git repository could work as a novel place to store this data.
Every time a temperature recording is made, the data can be committed to the repository and pushed to GitHub, then Alexa would be able to fetch the most recent recording from the raw content of the data file. It would also offer a full history of every temperature recording with delta compression for free.

The downside to storing my room temperature data in a public git repository is that, of course, it creates the security risk whereby a would-be burglar could study the temperature patterns and potentially discover when I am not in the room based on some kind of statistical model or machine learning algorithm.

I created a new GitHub user and [repository](https://github.com/raspberry-commits/bedroom-temperature-api), and added the public SSH key from the Pi to GitHub for commit access.

![GitHub Repo]({{ '/assets/rpi-temp-logger/github-repo.png' | relative_url }})

Here is the script to record the temperature data and push it to GitHub:

```python
import time
import datetime
import json
import Adafruit_DHT
import os

interval = int(os.getenv("INTERVAL", 120))
measurement = "rpi-dht22"
location = "bedroom"

# Temp sensor
sensor = Adafruit_DHT.DHT22
sensor_gpio = 4

try:
    while True:
        humidity, temperature = Adafruit_DHT.read_retry(sensor, sensor_gpio)
        humidity = round(humidity, 2)
        temperature = round(temperature, 2)
        iso = datetime.datetime.utcnow().strftime('%Y-%m-%dT%H:%M:%SZ')
        print("[%s] Temp: %s, Humidity: %s" % (iso, temperature, humidity))
        data = [
            {
                "measurement": measurement,
                "tags": {
                    "location": location,
                },
                "time": iso,
                "fields": {
                    "temperature": temperature,
                    "humidity": humidity
                }
            }
        ]
        f = open("/home/pi/bedroom-temperature-api/temperature.json", "w")
        f.write(json.dumps(data[0], sort_keys=True, indent=4))
        f.close()
        os.system("cd /home/pi/bedroom-temperature-api && "
                  "git add temperature.json "
                  "&& git commit -m '" + iso + "' "
                  "&& git push origin master")

        time.sleep(interval)

except KeyboardInterrupt:
    pass
```

The data structure can be posted to Influx DB, which is where I was originally storing the data before the SD card on the Pi crapped out on me.

This script can be run indefinitely in a screen:

```
screen -d -m python3 rpi-temp-sensor/temp-logger-to-gh.py
```

I added the command below to `/etc/rc.local` so that it runs on system boot.

```
sudo su - pi -c "screen -dm -S tempsensor python3 /home/pi/rpi-temp-sensor/temp-logger-to-gh.py"
```

### Creating the Alexa Skill


There are a lot of different ways to build an Alexa skill and it turns out to be a very vast and deep topic. I went down the route of adapting the simplest Hello World example to get things working, after all, I was only looking for Alexa to respond to one thing, I did not need a complex interaction model.

In the Alexa Developer console, I created a new custom skill called 'Rpi Temperature Sensor' and provisioned it with Alexa-Hosted Python backend resources.

Then I set the Skill Invocation Name (the name users say to invoke the skill) to 'room temperature'. This allows me to say "Alexa, room temperature" for my skill, and thus code, to be executed.

Everything else I left as default.

As mentioned earlier, the code is based upon the [Hello World](https://github.com/alexa/skill-sample-python-helloworld-classes/blob/master/lambda/py/hello_world.py) example.

```python
# -*- coding: utf-8 -*-

import logging
import ask_sdk_core.utils as ask_utils
import requests
from datetime import datetime


from ask_sdk_core.skill_builder import SkillBuilder
from ask_sdk_core.dispatch_components import AbstractRequestHandler
from ask_sdk_core.dispatch_components import AbstractExceptionHandler
from ask_sdk_core.handler_input import HandlerInput

from ask_sdk_model import Response

logger = logging.getLogger(__name__)
logger.setLevel(logging.INFO)


class LaunchRequestHandler(AbstractRequestHandler):
    """Handler for Skill Launch."""
    def can_handle(self, handler_input):
        # type: (HandlerInput) -> bool

        return ask_utils.is_request_type("LaunchRequest")(handler_input)

    def handle(self, handler_input):
        # type: (HandlerInput) -> Response
        r = requests.get("https://raw.githubusercontent.com/raspberry-commits/bedroom-temperature-api/master/temperature.json")
        data = r.json()

        speak_output = 'The temperature is %d degrees celcius (recorded %s)' % (data['fields']['temperature'], self.pretty_date(datetime.strptime(data['time'], '%Y-%m-%dT%H:%M:%SZ')))

        return (
            handler_input.response_builder
                .speak(speak_output)
                .response
        )

    def pretty_date(self, time):
        now = datetime.now()
        diff = now - time
        second_diff = diff.seconds
        day_diff = diff.days

        if day_diff < 0:
            return ''

        if day_diff == 0:
            if second_diff < 10:
                return "just now"
            if second_diff < 60:
                return str(int(second_diff)) + " seconds ago"
            if second_diff < 120:
                return "a minute ago"
            if second_diff < 3600:
                return str(int(second_diff / 60)) + " minutes ago"
            if second_diff < 7200:
                return "an hour ago"
            if second_diff < 86400:
                return str(int(second_diff / 3600)) + " hours ago"
        if day_diff == 1:
            return "Yesterday"
        if day_diff < 7:
            return str(day_diff) + " days ago"
        if day_diff < 31:
            return str(int(day_diff / 7)) + " weeks ago"
        if day_diff < 365:
            return str(int(day_diff / 30)) + " months ago"
        return str(int(day_diff / 365)) + " years ago"



class CatchAllExceptionHandler(AbstractExceptionHandler):
    def can_handle(self, handler_input, exception):
        # type: (HandlerInput, Exception) -> bool
        return True

    def handle(self, handler_input, exception):
        # type: (HandlerInput, Exception) -> Response
        logger.error(exception, exc_info=True)

        speak_output = "Sorry, I had trouble fetching the room temperature data. Please try again."

        return (
            handler_input.response_builder
                .speak(speak_output)
                .ask(speak_output)
                .response
        )


sb = SkillBuilder()
sb.add_request_handler(LaunchRequestHandler())
sb.add_exception_handler(CatchAllExceptionHandler())

lambda_handler = sb.lambda_handler()
```

To keep things as simple as possible, I stripped back the example and added my custom code to the `LaunchRequestHandler` class where the `handle` method is executed when the skill is launched.

The code fetches the temperature data from GitHub and extracts it alongside a human-understandable time since recording for Alexa to speak. E.g.: "The temperature is 18 degrees celcius (recorded 1 minute ago)".

I deployed this from the Alexa console without the need to publish to live. Considering I have no plans to make this public, deploying it to dev, to work on my Alexa alone is sufficient. 

Now I believe it is possible to use the Alexa Smart Home skills as a basis to make the integration cleaner. For example, if I say, "Alexa, what is the room temperature?", she gets confused and responds with "Bedroom doesn't support that". Ideally, Alexa would respond to that command using the Smart Home interaction model, but I decided building a full integration would take far more effort than I was willing to put in.

And that's it! A Raspberry Pi Temperature sensor logging to GitHub with a very hacky Alexa skill providing a voice interface.

