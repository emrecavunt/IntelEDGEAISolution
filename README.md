# Server Communications - Solution

Let's focus on MQTT first, and then FFmpeg.

## Additional Information

Note: Since you are given the MQTT Broker Server and Node Server for the UI, you need
certain information to correctly configure, publish and subscribe with MQTT.
- The MQTT port to use is 3001 - the classroom workspace only allows ports 3000-3009
- The topics that the UI Node Server is listening to are "class" and "speedometer"
- The Node Server will attempt to extract information from any JSON received from the MQTT server with the keys "class_names" and "speed"

## Running the App

First, get the MQTT broker and UI installed.

- `cd webservice/server`
- `npm install`
- When complete, `cd ../ui`
- And again, `npm install`

You will need *four* separate terminal windows open in order to see the results. The steps
below should be done in a different terminal based on number. You can open a new terminal
in the workspace in the upper left (File>>New>>Terminal).

1. Get the MQTT broker installed and running.
  - `cd webservice/server/node-server`
  - `node ./server.js`
  - You should see a message that `Mosca server started.`.
2. Get the UI Node Server running.
  - `cd webservice/ui`
  - `npm run dev`
  - After a few seconds, you should see `webpack: Compiled successfully.`
3. Start the ffserver
  - `sudo ffserver -f ./ffmpeg/server.conf`
4. Start the actual application.
  - First, you need to source the environment for OpenVINO *in the new terminal*:
    - `source /opt/intel/openvino/bin/setupvars.sh -pyver 3.5`
  - To run the app, I'll give you two items to pipe in with `ffmpeg` here, with the rest up to you:
    - `-video_size 1280x720`
    - `-i - http://0.0.0.0:3004/fac.ffm`

Your app should begin running, and you should also see the MQTT broker server noting
information getting published.

In order to view the output, click on the "Open App" button below in the workspace.

### MQTT

First, I import the MQTT Python library. I use an alias here so the library is easier to work with.

```
import paho.mqtt.client as mqtt
```

I also need to `import socket` so I can connect to the MQTT server. Then, I can get the
IP address and set the port for communicating with the MQTT server.

```
HOSTNAME = socket.gethostname()
IPADDRESS = socket.gethostbyname(HOSTNAME)
MQTT_HOST = IPADDRESS
MQTT_PORT = 3001
MQTT_KEEPALIVE_INTERVAL = 60
```

This will set the IP address and port, as well as the keep alive interval. The keep alive interval
is used so that the server and client will communicate every 60 seconds to confirm their
connection is still open, if no other communication (such as the inference statistics) is received.

Connecting to the client can be accomplished with:

```
client = mqtt.Client()
client.connect(MQTT_HOST, MQTT_PORT, MQTT_KEEPALIVE_INTERVAL)
```

Note that `mqtt` in the above was my import alias - if you used something different, that line
will also differ slightly, although will still use `Client()`.

The final piece for MQTT is to actually publish the statistics to the connected client.

```
topic = "some_string"
client.publish(topic, json.dumps({"stat_name": statistic}))
```

The topic here should match to the relevant topic that is being subscribed to from the other
end, while the JSON being published should include the relevant name of the statistic for
the node server to parse (with the name like the key of a dictionary), with the statistic passed
in with it (like the items of a dictionary).

```
client.publish("class", json.dumps({"class_names": class_names}))
client.publish("speedometer", json.dumps({"speed": speed}))
```

And, at the end of processing the input stream, make sure to disconnect.

```
client.disconnect()
```

### FFmpeg

FFmpeg does not actually have any real specific imports, although we do want the standard
`sys` library

```
import sys
```

This is used as the `ffserver` can be configured to read from `sys.stdout`. Once the output
frame has been processed (drawing bounding boxes, semantic masks, etc.), you can write
the frame to the `stdout` buffer and `flush` it.

```
sys.stdout.buffer.write(frame)
sys.stdout.flush()
```

And that's it! As long as the MQTT and FFmpeg servers are running and configured
appropriately, the information should be able to be received by the final node server,
and viewed in the browser.

To run the app itself, with the UI server, MQTT server, and FFmpeg server also running, do:

```
python app.py | ffmpeg -v warning -f rawvideo -pixel_format bgr24 -video_size 1280x720 -framerate 24 -i - http://0.0.0.0:3004/fac.ffm
```

This will feed the output of the app to FFmpeg.
