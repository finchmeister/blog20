---
layout: post
title: Visualising Raspberry Pi Temperature Sensor Monitoring (Part 2)
description: How to import sensor data into InfluxDB and visualise with Grafana
summary: How to import sensor data into InfluxDB and visualise with Grafana
tags: [tech, python, raspberrypi, docker]
---

Following on from [Part 1](/2020/10/31/raspberry-pi-temp-logger), being able to visualise the temperature data was an important goal.
Inspired by [this](https://www.definit.co.uk/2018/07/monitoring-temperature-and-humidity-with-a-raspberry-pi-3-dht22-sensor-influxdb-and-grafana/) blog post, I build a very similar setup whereby the temperature data is piped into InfluxDB and the

Grafana is typically used as a query and visualisation platform for software metrics but it also works well for IoT data.


![Dashboard Screenshot]({{ '/assets/rpi-temp-logger/grafana-dashboard.png' | relative_url }})

Originally I was running the temperature logger, InfluxDB and Grafana setup on the same Raspberry Pi but it turns out SD cards are not very reliable and are prone to corruption after time.
I lost a lot of data.

## Setting up the Raspberry Pi

I used another, more recent and powerful model 4 Raspberry Pi to run all this.
I found the older Pi often struggled with the load.

Again I set the Pi up in headless mode with Raspbian Lite.

I installed Docker with:

```
curl -sSL https://get.docker.com/ | sh
sudo usermod -aG docker $USER
```

Setting up InfluxDB and Grafana with Docker is very simple. All it takes is this `docker-compose.yml` file:

```yaml
version: '2'

services:
  influxdb:
    image: influxdb
    restart: always
    environment:
      INFLUXDB_DB: "sensor_data"
      INFLUXDB_USER: "rpi"
      INFLUXDB_USER_PASSWORD: "rpi"
    volumes:
      - influxdb_data:/var/lib/influxdb
    ports:
      - "8086:8086"
  grafana:
    image: grafana/grafana
    restart: always
    depends_on:
      - influxdb
    volumes:
      - grafana-storage:/var/lib/grafana
    ports:
      - "3000:3000"

volumes:
  influxdb_data:
  grafana-storage:
```

The InfluxDB configuration here is very important. I define the database name, user and user password in the compose file which will all be needed later when pushing data into the DB, and for setting up the data source in Grafana.
I set up volumes so that the data is persisted, and exposed the ports so that we can talk to the containers from outside.

To get InfluxDB and Grafana started:
```
docker-compose up -d
```

Grafana can now be accessed on port 3000 on the Pi's IP.


### Pushing the data into InfluxDB

If the temperature recording was being made on the same Pi it would be very simple to push the data directly into InfluxDB. Instead as I am pushing the data to GitHub, I needed a separate script to fetch the recent recordings from there and push it into InfluxDB.
It is definitely a bit weird to be using GitHub as an intermediary, but if it's stupid and it works, it ain't stupid... :shrug:

I wrote this Python script to go through all the commits from the temperature data repository and push them into InfluxDB.
It works as a complete backup restore but will also incrementally push data into InfluxDB as new commits are pushed to the repository.

```python
#! /usr/bin/python
from influxdb import InfluxDBClient
import os
import json

host = "localhost"
port = 8086
user = "rpi"
password = "rpi"
db_name = "sensor_data"
tempdata_path = "/home/pi/bedroom-temperature-api"
client = InfluxDBClient(host, port, user, password, db_name)
checkpoint_path = os.path.dirname(os.path.abspath(__file__)) + "/checkpoint.txt"


def reached_checkpoint(commit_hash):
    return commit_hash == get_checkpoint()


def save_checkpoint(commit_hash):
    f = open(checkpoint_path, "w")
    f.write(commit_hash)
    f.close()


def get_checkpoint():
    f = open(checkpoint_path, "r")
    return f.read().strip()


def get_data(commit_hash):
    stream = os.popen("cd " + tempdata_path + " && git show " + commit_hash + ":temperature.json | cat")
    return [json.loads(stream.read())]


def write_data(data):
    if data[0]["fields"]["humidity"] is None or data[0]["fields"]["temperature"] is None:
        return
    client.write_points(data)


def pull_latest():
    os.popen("cd " + tempdata_path + " && git pull")


def get_commits_in_reverse_chronological():
    stream = os.popen("cd " + tempdata_path + " && git rev-list master")
    return stream.readlines()


def run():
    pull_latest()

    next_checkpoint = None
    for line in get_commits_in_reverse_chronological():
        commit_hash = line.strip()

        if next_checkpoint is None:
            next_checkpoint = commit_hash

        if reached_checkpoint(commit_hash) is True:
            break

        data = get_data(commit_hash)
        write_data(data)

    save_checkpoint(next_checkpoint)


run()
```

As it loops through the commits in reverse-chronological, I store the first commit hash I see as a checkpoint, then I push each data point to the DB until I reach a previous checkpoint, or write every data point (as on first run).
Future executions only write the deltas.

I set this up as a cron job to run every minute from `crontab -e`:

```
* * * * * /usr/bin/python3 /home/pi/rpi-temp-sensor/recover-git-data/restore_db.py >/dev/null 2>&1
```

### Setting Up Grafana

Grafana needs InfluxDB to be set up as a data source.

As I am running this in Docker, the host name of InfluxDB in the eyes of the docker network is `influxdb`, and the port we are using is `8086`.
`http://influxdb:8086`

The InfluxDB Details match what I configured in the docker-compose file:

Database: `sensor_data`  
User: `rpi`  
Password: `rpi`  

With the data source configured, I created a new dashboard and added a panel.

![Line panel setup]({{ '/assets/rpi-temp-logger/temp-line-panel-setup.png' | relative_url }})

[This guy](https://www.definit.co.uk/2018/07/monitoring-temperature-and-humidity-with-a-raspberry-pi-3-dht22-sensor-influxdb-and-grafana/) nailed the graphs so I basically copied his configuration and also set up a gauge visualisation.

![Guage panel setup]({{ '/assets/rpi-temp-logger/temp-guage-panel-setup.png' | relative_url }})

That's the InfluxDB/Grafana setup covered! 

Now I found reliability an issue with the temperature logger. There were a couple of incidents were it stopped taking measurements, either because it did not recover from a power outage, the SD card blew up.
Part 3 goes into detail of how I set up a monitor to check that the temperature sensor was working.