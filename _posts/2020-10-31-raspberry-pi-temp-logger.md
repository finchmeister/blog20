---
layout: post
title: Alexa Raspberry Pi Temperature Sensor
description: How to set up a Raspberry Pi Temperature Sensor
summary: How to set up a Raspberry Pi Temperature Sensor
tags: [tech]
---

In this article I will share how I hacked together an Alexa connected room temperature sensor using a Raspberry Pi and a GitHub repository. 

> "Alexa, room temperature?"
>
> **The temperature is 18 degrees celcius (recorded 1 minute ago)**

### Setting up the Pi

For this project, I started with an old Raspberry pi 2 model with a usb wifi dongle.
I used a DHT 22 temperature sensor.

The temperature sensor is connected to the pi like so:

PHOTO

```
Pin 1 - 3.3v power (+)
Pin 4 - GPIO one-wire data out
Pin 9 - Ground (-)
```

I did the usual prep work to get remote SSH access to the Pi via Wifi

- Use Raspberry Pi Imager to prepare the SD card with Raspbian
- Add SSH file to the root
- Create a `wpa_supplicant.conf` file with the wifi information

### Recording the Temperature

To access the temperature data in a script, I used the [Python Adafruit DHT library](https://github.com/adafruit/Adafruit_Python_DHT) which I installed with:

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

### Making the data accessible to Alexa

Unsurprisingly, Alexa needs access to the temperature data for her to voice it back. I could have exposed the Pi to the web as a web server for Alexa to access the data directly but that involves a lot of faff and messing around with router configuration.

Instead, I realised that a public git repository could work as a novel place to store this data.
Every time a temperature recording is made, the data could be committed and pushed to GitHub, then Alexa would be able to fetch the most recent recording from the raw content of the data file. It would also offer a full history of every temperature recording with delta compression for free.

Now I appreciate this is not the most secure method...

To set that up I created a new GitHub user and created the repository and added the public SSH key from the Pi to GitHub for commit access.

https://github.com/raspberry-commits/bedroom-temperature-api



Then wrote a script to record the temperature data and push it to GitHub:

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

The data structure follows that to satisfy Influx DB which is where I was originally posting the data, more details on that in a separate blog post.

This script can be run indefinitely in a screen:

```
screen -d -m python3 rpi-temp-sensor/temp-logger-to-gh.py
```

I added the command above to `/etc/rc.local` so that it runs on system boot.

### Creating the Alexa Skill


There are a lot of different ways to build an Alexa skill and it turns out to be a very vast and deep topic. I went down the route of adapting the simplest Hello World example to get things working. After all I only need Alexa to respond to one thing. 

In the Alexa Developer console, I created a new custom skill
 called Rpi Temperature Sensor with Alexa-Hosted Python backend resources.

Then I set the Skill Invocation Name (the name users say to invoke the skill) to 'room temperature'. This allows me to say "Alexa, room temperature" for my skill, and then code, to be executed.

Everything else I left as default.

Now the code I am currently using is based upon the [Hello World](https://github.com/alexa/skill-sample-python-helloworld-classes/blob/master/lambda/py/hello_world.py) example as mentioned earlier.

```python
# -*- coding: utf-8 -*-

# This sample demonstrates handling intents from an Alexa skill using the Alexa Skills Kit SDK for Python.
# Please visit https://alexa.design/cookbook for additional examples on implementing slots, dialog management,
# session persistence, api calls, and more.
# This sample is built using the handler classes approach in skill builder.
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


class CancelOrStopIntentHandler(AbstractRequestHandler):
    """Single handler for Cancel and Stop Intent."""
    def can_handle(self, handler_input):
        # type: (HandlerInput) -> bool
        return (ask_utils.is_intent_name("AMAZON.CancelIntent")(handler_input) or
                ask_utils.is_intent_name("AMAZON.StopIntent")(handler_input))

    def handle(self, handler_input):
        # type: (HandlerInput) -> Response
        speak_output = "Goodbye!"

        return (
            handler_input.response_builder
                .speak(speak_output)
                .response
        )


class SessionEndedRequestHandler(AbstractRequestHandler):
    """Handler for Session End."""
    def can_handle(self, handler_input):
        # type: (HandlerInput) -> bool
        return ask_utils.is_request_type("SessionEndedRequest")(handler_input)

    def handle(self, handler_input):
        # type: (HandlerInput) -> Response

        # Any cleanup logic goes here.

        return handler_input.response_builder.response


class IntentReflectorHandler(AbstractRequestHandler):
    """The intent reflector is used for interaction model testing and debugging.
    It will simply repeat the intent the user said. You can create custom handlers
    for your intents by defining them above, then also adding them to the request
    handler chain below.
    """
    def can_handle(self, handler_input):
        # type: (HandlerInput) -> bool
        return ask_utils.is_request_type("IntentRequest")(handler_input)

    def handle(self, handler_input):
        # type: (HandlerInput) -> Response
        intent_name = ask_utils.get_intent_name(handler_input)
        speak_output = "You just triggered " + intent_name + "."

        return (
            handler_input.response_builder
                .speak(speak_output)
                # .ask("add a reprompt if you want to keep the session open for the user to respond")
                .response
        )


class CatchAllExceptionHandler(AbstractExceptionHandler):
    """Generic error handling to capture any syntax or routing errors. If you receive an error
    stating the request handler chain is not found, you have not implemented a handler for
    the intent being invoked or included it in the skill builder below.
    """
    def can_handle(self, handler_input, exception):
        # type: (HandlerInput, Exception) -> bool
        return True

    def handle(self, handler_input, exception):
        # type: (HandlerInput, Exception) -> Response
        logger.error(exception, exc_info=True)

        speak_output = "Sorry, I had trouble doing what you asked. Please try again."

        return (
            handler_input.response_builder
                .speak(speak_output)
                .ask(speak_output)
                .response
        )

# The SkillBuilder object acts as the entry point for your skill, routing all request and response
# payloads to the handlers above. Make sure any new handlers or interceptors you've
# defined are included below. The order matters - they're processed top to bottom.


sb = SkillBuilder()

sb.add_request_handler(LaunchRequestHandler())
sb.add_request_handler(CancelOrStopIntentHandler())
sb.add_request_handler(IntentReflectorHandler()) # make sure IntentReflectorHandler is last so it doesn't override your custom intent handlers

sb.add_exception_handler(CatchAllExceptionHandler())

lambda_handler = sb.lambda_handler()
```

To keep things as simple as possible, I stripped back the example and added my custom code to the `LaunchRequestHandler` class where the `handle` method is executed when the skill is launched.

The code fetches the temperature data from GitHub and extracts it into a phrase for Alexa to speak with a human understandable time since recording metric.

I deployed this from the Alexa console without the need to publish to live. Considering I have no plans to make this public, deploying it to dev, to work on my Alexa alone is sufficient. 

And that's it! A Raspberry Pi Temperature sensor logging to GitHub with a very hacky Alexa skill providing a voice interface.