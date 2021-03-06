
# de0gee-data

[![travis](https://travis-ci.org/schollz/find3/server/main.svg?branch=master)](https://travis-ci.org/schollz/find3/server/main) 
[![go report card](https://goreportcard.com/badge/github.com/schollz/find3/server/main)](https://goreportcard.com/report/github.com/schollz/find3/server/main) 
[![coverage](https://img.shields.io/badge/coverage-94%25-brightgreen.svg)](https://gocover.io/github.com/schollz/find3/server/main)
[![godocs](https://godoc.org/github.com/schollz/find3/server/main?status.svg)](https://godoc.org/github.com/schollz/find3/server/main) 

<h3 class="section-head" id="intro"><a href="#intro">Introduction</a></h3>

The datastore server is a Go server that allows you to store sensor data. In production this server sits behind an authentication server that first authenticates users before allowing access. You can also run the server publicly, without authentication.

<h3 class="section-head" id="sensor"><a href="#sensor">Sensor data</a></h3>

The main element of the datastore server is the **SensorData**. **SensorData** is sent to the datastore server via JSON. The most basic JSON for **SensorData** is the following:

```json
{
    "t":1514034330040,
    "f":"fido and friends",
    "d":"fido's phone",
    "l":"dog house",
    "s":{     
    }
 }
```

The keys in this **SensorData** are shorthand (single characters) to cut down on bandwidth for sending/receiving JSON. They characters are "t" for "timestamp", "f" for "family", "d" for "device", "l" for "location" and "s" is for the "sensor readings." 

A timestamp ("t") uniquely identifies a piece of **SensorData**, as it is the UNIX epoch time *in milliseconds*, making it highly unlikely for clashes to exist. 

The family ("f") is the group in which the device belongs. A family can have many devices associated with it (your phone, your computer, your dog's collar), but each device can only be associated with one family.

The device ("d") uniquely identifies the current device in that particular family.

The location ("l") classifies the location of the current **SensorData**. This is optional, and it is used for classifying the location in preparation for machine learning.

The sensor readings ("s") here is empty. The sensor readings in this most basic JSON are blank, as they are optional, although they are the most important part of the **SensorData**. Sensor readings are added to the JSON as maps of sensor data. For example, if you are taking WiFi data from access points, you would format your sensor readings as:

``` 
"wifi": {
  "aa:bb:cc:dd:ee":-20,
  "ff:gg:hh:ii:jj":-80
}
```

The first key explains the sensor type ("wifi") and the key and values inside the map explain the sensor name (a MAC address) and the value (the signal dBm). The same format is followed for *any kind of sensor*. For example, here is the sensor readings formatted for accelerometer data:

```
"accelerometer": {
  "x":-1.11,
  "y":2.111,
  "z":1.23   
}
```

In this case the sensor name is the coordinate and the value is the acceleration in that direction. So in the end, if your phone collects a lot of data you will end up sending the following **SensorData** JSON to the datastore server:

```json
{
    "t":1514034330040,
    "f":"fido and friends",
    "d":"fido's phone",
    "l":"dog house",
    "s":{
         "wifi":{
                "aa:bb:cc:dd:ee":-20,
                "ff:gg:hh:ii:jj":-80
         },
         "bluetooth":{
                "kk:ll:mm:nn:oo":-42,
                "pp:qq:rr:ss:tt":-50        
         },
         "temperature":{
                "sensor1":12,
                "sensor2":20       
         },
         "accelerometer":{
                "x":-1.11,
                "y":2.111,
                "z":1.23   
         }      
    }
 }
```

<h3 class="section-head" id="find"><a href="#find">FIND compatibility</a></h3>

The datastore server does have [FIND compatibility](https://www.internalpositioning.com/api/#post-learn). That is, the POST /track and POST /learn routes are still available, where the payload that is sent is the same as in FIND:

```json
{
   "group":"some group",
   "username":"some user",
   "location":"some place",
   "timestamp":12309123,
   "wifi-fingerprint":[
      {
         "mac":"AA:AA:AA:AA:AA:AA",
         "rssi":-45
      },
      {
         "mac":"BB:BB:BB:BB:BB:BB",
         "rssi":-55
      }
   ]
}
```

However, FIND only supports WiFi, so you cannot use these routes for sending other kinds of data.

<h3 class="section-head" id="quick-start"><a href="#quick-start">Quick start</a></h3>

First make sure that [you have installed Go](/docs/quick-start/#install-go).

## API 

Starting the datastore server gives you several endpoints to insert, delete, or pull information.

<h3 class="section-head" id="post-slash"><a href="#post-slash"><code>POST / (insert data)</code></a></h3>

**Parameters**:

Requires JSON of the sensor data, e.g. 
```json
{
    "t":1514034330040,
    "f":"fido and friends",
    "d":"fido's phone",
    "l":"dog house",
    "s":{
         "wifi":{
                "aa:bb:cc:dd:ee":-20,
                "ff:gg:hh:ii:jj":-80
         },
         "bluetooth":{
                "kk:ll:mm:nn:oo":-42,
                "pp:qq:rr:ss:tt":-50        
         },
         "temperature":{
                "sensor1":12,
                "sensor2":20       
         },
         "accelerometer":{
                "x":-1.11,
                "y":2.111,
                "z":1.23   
         }      
    }
 }
```

**Response**:

```json
{
    "success": true,
    "message": "inserted sensor data",
}
```


### Testing


```
# Start machine learning server
cd $GOPATH/de0gee/de0gee-ai/src
export FLASK_DEBUG=1 && export FLASK_APP=server.py && flask run --debugger --port 8002

# Load machine learning data
cd $GOPATH/de0gee/de0gee-ai/testing
./learn.sh

# Test classification
http localhost:8002/classify < testdb_single_rec.json

# Start datastore server
cd $GOPATH/schollz/find3/server/main
go build && ./de0gee-data

# Test getting the classification of the latest location
http --json GET localhost:8003/location family=testdb device=zack2@gmail.com
```

Supervisord file:

```
[supervisorctl]

[supervisord]

[program:de0gee-data]
directory=/home/zns/go/src/github.com/schollz/find3/server/main
command=de0gee-data
stdout_logfile: /home/zns/go/src/github.com/schollz/find3/server/main.std.log
stdout_logfile: /home/zns/go/src/github.com/schollz/find3/server/main.err.log

[program:de0gee-ai]
directory=/home/zns/go/src/github.com/de0gee/de0gee-ai/src
environment = 
    FLASK_APP=server.py,
    FLASK_DEBUG=1
command=/usr/local/bin/flask run --debugger --port 8002
stdout_logfile: /home/zns/go/src/github.com/de0gee/de0gee-ai.std.log
stdout_logfile: /home/zns/go/src/github.com/de0gee/de0gee-ai.err.log
```

http://www.steves-internet-guide.com/install-mosquitto-linux/

MOSQUITTO

```
# bootstrap
cd $GOPATH/src/github.com/schollz/find3/server/main/src/mqtt
go test 
pkill -9 mosquitto
mosquitto -c mosquitto_config/mosquitto.conf -d

# this should allow you to subscribe (change password though)
http POST localhost:8003/mqtt family=testdb
mosquitto_sub -h localhost -p 1883 -u testdb -P de7r3 -t 'testdb/#'

# labs should see this
mosquitto_pub -u zack -P 1234 -t 'labs/location' -m 'hello'

# labs should not see this
mosquitto_pub -u zack -P 1234 -t 'someother' -m 'hello'

# if you start labs with -t '#' it should not see anything, but admin can
```

Starting up everything

```
cd $GOPATH/src/github.com/schollz/find3/server/main/src/mqtt && /usr/sbin/mosquitto -c mosquitto_config/mosquitto.conf -d
cd $GOPATH/src/github.com/schollz/find3/server/main && ./de0gee-data

# for debugging
cd $GOPATH/src/github.com/de0gee/de0gee-ai/src && export FLASK_APP=server.py && export FLASK_DEBUG=1 && flask run --debugger --port 8002

# for production
cd $GOPATH/src/github.com/de0gee/de0gee-ai/src && gunicorn server:app -b 0.0.0.0:8002 -w 8
```

Ideas: use one-time-pass API keys for accessing pieces of the AI server or the datastore server directly (mainly things like websockets).

Overview of program: https://swimlanes.io/u/S1r5LsmmG:

Title: Uploading Sensor Data

_: **Data collection** 

Note de0gee, Auth: You can collect data via the de0gee device.

de0gee -> de0gee: Collect **Sensor Data**


de0gee -> User Phone: **Sensor Data** via bluetooth

User Phone -> Auth: `POST /data`: **Sensor Data** + User IOT stuff API Key via HTTPS


Note de0gee, Auth: Or, you can collect data via the phone.

User Phone -> User Phone: Collect **Sensor Data**


User Phone -> Auth: `POST /data`: **Sensor Data** + User API Key via HTTPS


_: **Data storage** 

Auth -> Auth: Validate user using API key

Auth -->> User Phone: `Response`: _Return if invalid_

Auth -> Data: `POST /data`: **Sensor Data** via LAN




Data -> Data: Store **Sensor Data**

Data -> Auth: `Response`: success/failure

Auth -> User Phone: `Response`: success/failure 

_: **Data analysis** (via Goroutine)

Data -> AI: `POST /classify`: **Sensor Data** via LAN

AI -> AI: Classify **Sensor Data** to get Location Data

AI -> Data: `Response`: **Location Data**

Data -> Data: Store **Location Data**

Data -->> User IOT stuff: **Location Data** via authenticated MQTT

Data -> Auth: `POST /location`: **Location data** via LAN


Auth -->> User Browser: **Location Data** via secure websockets

![Flow](https://user-images.githubusercontent.com/6550035/34440974-2002dd90-ec76-11e7-89fb-481e4fadea28.png)


## Debugging pipelines

Terminal1:

Note the server address: http://Y:8003
```
cd $GOPATH/src/github.com/schollz/find3/server/main && make dev1 | grep -v 'db.go'
```

Terminal2:

```
cd $GOPATH/src/github.com/de0gee/de0gee-ai && make
```

Raspberry Pi, name X

```
while true; do; sudo ./scanner_arm -i wlx98ded0128a34 -debug -device X -family test5 -reverse -server http://Y:8003 -no-modify -scantime 8 ; done;
```

Do some learning with your phone. Get your phone's mac address, `ma:ca:dd:rr:es:s1`. Set the timestamp to `1` to make changes for learning.

```
http POST localhost:8003/reverse t:=1 f=test5 d=ma:ca:dd:rr:es:s1 l=LOCATION
```

You can add other phones in other locations to do learning:

```
http POST localhost:8003/reverse t:=1 f=test5 d=ma:ca:dd:rr:es:s2 l=LOCATION
```

When you are done with learning, turn off the learning by switching off the location *for each device*:


```
http POST localhost:8003/reverse t:=1 f=test5 d=ma:ca:dd:rr:es:s1
http POST localhost:8003/reverse t:=1 f=test5 d=ma:ca:dd:rr:es:s2
```

Calibrate your device using

```
http POST localhost:8003/calibrate family=test5
```

And then see the location of any device using

```
http GET localhost:8003/location family=test5 device=ma:ca:dd:rr:es:s1
```

Get just the best guess using:

```
http GET localhost:8003/location family=test5 device=ma:ca:dd:rr:es:s1 | jq .analysis.best_guess
```
