---
layout: post
title: Monitoring Raspberry Pi Temperature Sensor Monitor (Part 3)
description: How to setup a simple AWS Lambda function to monitor the temperature logger
summary:  How to setup a simple AWS Lambda function to monitor the temperature logger
tags: [tech, python, AWS]
---

The Raspberry Pi temperature monitoring setup I have has run into a few issues over time.
It has broken for a variety of different reasons such as a corrupted SD card, a failing script, a failure to recover after a power outage and so on...

I do not know it has broken until I look at the dashboard or ask Alexa what the room temperature is, so for a bit of fun I thought I would set up a monitor to check the temperature monitor.

The idea was to set up an AWS Lambda function that would hourly check the most recent temperature recording and send a push notification to my phone if the recording had not been made in the last hour.

[Pushbullet](https://www.pushbullet.com/) is a really nice service that allows you to send push notifications to your phone for free via a simple API call. All you need to do is create an account, install the app on your phone and generate an API Access token from within the web interface.
From there you can make an API request from whatever language to the `/v2/pushes` endpoint and receive the notification on your phone.

With the notification mechanism in place, I wrote a script that fetches the latest temperature recording from GitHub and compares it with the current time.
If the recording is over an hour old, something is wrong, so it will make the API request to Pushbullet to send the notification.
For simplicity and to prevent the need for a datastore, I do not store whether the notification for this incident has been sent. To prevent a continuous barrage of notifications, if the recording is over 3 hours old it will not send any more notifications.
As this script will run every hour, at worse I will get 2 notifications, much better than getting one every hour if I am away and cannot fix it.

Here is the script:

```python
import urllib3
import json
from datetime import datetime, timedelta
import os

urllib3.disable_warnings()
http = urllib3.PoolManager()


def lambda_handler(*args, **kwargs):
    monitor()


def get_last_recording():
    r = http.request("GET", "https://raw.githubusercontent.com/raspberry-commits/bedroom-temperature-api/master/temperature.json")
    data = json.loads(r.data.decode('utf-8'))

    return datetime.strptime(data['time'], '%Y-%m-%dT%H:%M:%SZ')


def send_notification_via_pushbullet(title, body):
    """ Sending notification via pushbullet.
        Args:
            title (str) : title of text.
            body (str) : Body of text.
    """
    data_send = {"type": "note", "title": title, "body": body}

    access_token = os.getenv("PUSHBULLET_ACCESS_TOKEN", "")

    if access_token == "":
        raise Exception("PUSHBULLET_ACCESS_TOKEN env not set")

    resp = http.request(
        "POST",
        "https://api.pushbullet.com/v2/pushes",
        body=json.dumps(data_send),
        headers={'Authorization': 'Bearer ' + access_token, 'Content-Type': 'application/json'}
    )

    if resp.status != 200:
        raise Exception('Something wrong sending notification')

    print('Notification sent')


def monitor():
    last_recording = get_last_recording()
    one_hour_ago = datetime.now() - timedelta(hours=1)
    three_hours_ago = datetime.now() - timedelta(hours=3)
    if last_recording > one_hour_ago:
        print("All good")
        return

    print("Not good")
    if three_hours_ago > last_recording:
        print("Already alerted")
        return

    print("Sending notification")
    send_notification_via_pushbullet("Temperature Sensor", "It is f*cked mate")


def is_running_in_lambda():
    return os.getenv("AWS_LAMBDA_FUNCTION_NAME", "") != ""


if is_running_in_lambda() is False:
    print("Running outside of Lambda")
    monitor()
```

Providing the `PUSHBULLET_ACCESS_TOKEN` environment variable has been set, this script can be run both locally and on AWS Lambda.

Note the `lamdba_handler` function, this is required by AWS Lambda, and will be called when the lamdba is invoked.

I used the `urllib3` http client library over `requests` as it is available in the AWS Lambda runtime without the need for additional build steps.

### Setting up the AWS Lambda

The great thing about lamdas is that they are cheap, simple and super easy to setup for small things.


For this mini-project, I created a new function from the AWS Lambda web interface with the Python 3.8 runtime.

I took the following steps to get it up and running:

1. Copy & pasted the Python code into the function code section of the web UI with the file name `monitor.py` and deployed the function
2. Added the `PUSHBULLET_ACCESS_TOKEN` environment variable
3. Configured the runtime Handler to be: `monitor.lambda_handler`

From here, I was able to test the lambda in the web UI with a dummy event.

To get the lamdba running every hour, I added an EventBridge trigger that runs on a schedule.

![Eventbridge trigger]({{ '/assets/rpi-temp-logger/aws-lambda-eventbridge-trigger.png' | relative_url }})

To make deployment easier, I added a make command that zips the monitor file and updates the function code via the AWS cli:

```
update-monitor-lambda:
	cd temp-logger-monitor; zip ../my-deployment-package.zip monitor.py
	aws lambda update-function-code --function-name temp-sensor-monitor --zip-file fileb://my-deployment-package.zip | cat
```

Future updates can be deployed with: 
```
make update-monitor-lambda
```

### Summary

Being able to run a script on a schedule and send push notifications for free is a powerful idea.