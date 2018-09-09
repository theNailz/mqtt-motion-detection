# MQTT motion detection

This project aims to provide an open source, DIY version for building your own motion detection sensor, for usage with Domoticz or any other MQTT implementation.

Source: [https://thenailz.github.io/mqtt-motion-detection/](https://thenailz.github.io/mqtt-motion-detection/)

## Requirements

### Hardware

- [NodeMCU](http://s.click.aliexpress.com/e/b4eIji0Y)
- [PIR sensor](http://s.click.aliexpress.com/e/NZE7tdI)

### Software

- [Arduino IDE](https://www.arduino.cc/en/Main/Software)
- [MQTT server](http://mqtt.org/)
- [Node-Red](https://nodered.org/)
- [Domoticz](http://www.domoticz.com/)

### Flow

The data flow for this project will look something like this:

```
PIR sensor > ESP > MQTT > Node-Red > MQTT > Domoticz variable
```

The ESP will read the PIR sensor status, and send a message to the MQTT servers. Node Red will consume this message and send Domoticz a variable update message to set a variable to `1`. After a period of inactivity, Node Red will send another update to Domoticz, setting a variable to `0`.

## Step 1: Building the hardware

The PIR sensor has three pins, **Vin**, **Gnd** and **Out**. Connect **Vin** to +5V, **Gnd** to ground and the **Out** pin to D7 on your NodeMCU. Connect the NodeMCU to the 5V power source.

You could also power the PIR sensor by connecting the ESP's **Vin** pin to the PIR sensor's **Vin** pin.

## Step 2: Software for the ESP8266

Start up the Arduino IDE, configure it for usage with an ESP8266, and load up the following sketch:

```c
#include <ESP8266WiFi.h>
#include <PubSubClient.h>

const char* ssid = "SSID";
const char* password = "SuperSecretPassword";
const char* mqtt_server = "192.168.1.200";
const char* device_name = "kitchen";

String topic = "presence/" + String(device_name);

int inputPin = D7;
int lastPirValue = LOW;

WiFiClient espClient;
PubSubClient client(espClient);

void setup_wifi() {
  delay(10);
  // We start by connecting to a WiFi network
  Serial.println();
  Serial.print("Connecting to ");
  Serial.println(ssid);

  WiFi.begin(ssid, password);

  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }

  randomSeed(micros());

  Serial.println("");
  Serial.println("WiFi connected");
  Serial.println("IP address: ");
  Serial.println(WiFi.localIP());
}

void reconnect() {
  // Loop until we're reconnected
  while (!client.connected()) {
    Serial.print("Attempting MQTT connection...");
    // Create a random client ID
    String clientId = "ESP8266Client-";
    clientId += String(random(0xffff), HEX);
    // Attempt to connect
    if (client.connect(clientId.c_str())) {
      Serial.println("connected");
    } else {
      Serial.print("failed, rc=");
      Serial.print(client.state());
      Serial.println(" try again in 5 seconds");
      // Wait 5 seconds before retrying
      delay(5000);
    }
  }
}

void setup() {
  Serial.begin(115200);
  setup_wifi();
  client.setServer(mqtt_server, 1883);

  pinMode(inputPin, INPUT);
}

void loop() {
  if (!client.connected()) {
    reconnect();
  }
  client.loop();

  int pirValue = digitalRead(inputPin);
  if (pirValue != lastPirValue) {
    if (pirValue == HIGH) {
      String message = String(pirValue);
      client.publish(topic.c_str(), message.c_str());
    }
    Serial.println(pirValue);
    lastPirValue = pirValue;

    delay(30 * 1000);
  }
}
```

Change the **ssid**, **password**, **mqtt_server** and **device_name** to suit your needs. Upload this code to your ESP and check the serial monitor for debug information. Make sure it's connecting to your WiFi and is detecting motion.

Note: It can take a couple of minutes for the PIR sensor to *warm up* and start processing signals. You can tune the sensitivity and delay by turning the knobs on your sensor.

## Step 3: Configuring Domoticz

When the sensor detects movement, a variable will be set in Domoticz. When there is movement, it will be set to `1` and when there is no movement detected for a specified time, it will be set to `0`.

### Configure Domoticz for MQTT

Visit [this page](https://www.domoticz.com/wiki/MQTT) for setting up Domoticz/MQTT communication.

### Set up a variable

First, we need to create a User Variable in Domoticz. To do so, go to Setup > More options > User variables. You can give it any name, set type to `Integer` and give it an initial value of `0`. I called mine `MOTION_KITCHEN`. Make a note of the `idx` value for your newly added variable.

### Sunset

If you'd like to turn on lights only between sunset and sunrise, you'd also need a dummy switch for this. Go to Setup > Hardware, add a Dummy device if you don't have one already, and add a Virtual Sensor to it. Give it the name `Sun` and type `Switch`. Now go to the Devices tab, and find your added device. Click *Timers* and add a `After Sunrise = Off` and `Before Sunset = On` timer. I like my devices to overlap the sunrise and sunset by 15 minutes.

This switch will be your status indicator of the current "sun status". When the sun is up, the switch is on, and when the sun is down, the switch goes off.

**Optional:** Edit the switch to make it `Protected` to accidental manual switching.

## Step 4: Configuring Node-RED

Since the PIR sensor only triggers when it sees movement, we need to reset the signal when there hasn't been any movement in the last couple of minutes. To do so, we'll be using Node Red to process the trigger, translate it to a message Domoticz understands, and reset the trigger after a set period of inactivity.

### Node Red plugins

This flow needs an additional plugin. Install the following plugins for Node Red through the top right menu > Manage palette:

- node-red-contrib-key-value-store

### Flow

I'll walk you through the flow on a node-by-node basis.

#### Node 1: MQTT in

The first node will be a MQTT consumer which will receive all messages from the ESP.

- Server: add/select your server
- Topic: `motion/#`
- QoS: default
- Name: *blank*

#### Node 2: switch

We will filter all values which are not `1`, to make sure we're not triggering on other values.

- Name: blank
- Property: `msg.payload`
- Rule 1: `==` `string ` `1`

Connect this node to Node 1.

#### Node 3.1: Function

This node will set a timestamp on the payload to set the latest trigger timestamp.

- Name: `Set timestamp`

- Function: 

  ```javascript
  msg.payload = parseInt((new Date()).getTime() / 1000);
  return msg;
  ```

Connect this node to Node 2.

#### Node 4.1: key-value-write

This node will write the timestamp to disk.

- Store: add/select one
- Action: `set`
- Key: *blank*
- Value: *blank*
- Name: `Persist timestamp`

Connect this node to Node 3.1.

#### Node 5.1: function

This node will set a new payload for Domoticz, to update our created User Variable.

- Name: `Enable presence`

- Function:

  ```javascript
  msg.payload = {
      "command": "setuservariable",
      "idx": 2,
      "value": 1
  };
  return msg;
  ```

Make sure to update the `idx` value with the `idx`value of your User Variable. The `command` and `value` should not be changed!

Connect this node to Node 4.1.

#### Node 6: MQTT out

This node will publish the messages sent to it.

- Server: select your server from step 1.
- Topic: `domoticz/in`
- QoS: *blank*
- Retain: *blank*
- Name: blank

Connect this node to Node 5.1.

#### Node 3.2: delay

This node (connected to Node 2) will delay the "turn off" signal to Domoticz.

- Action: `Delay each message` and `Fixed delay`
- For: `15 minutes`
- Name: *blank*

#### Node 4.2: key-value-read

This node (connected to node 3.2) will read the last stored timestamp and add it to the message.

- Store: select your store from Node 4.1.
- Key: *blank*
- Name: *blank*

#### Node 5.2: function

This node (connected to 4.2) adds a property to the message object, indicating if it should turn off or not.

- Name: `Add shouldTurnOff`

- Function:

  ```javascript
  var now = parseInt((new Date()) / 1000);
  
  // TTL (seconds) should be a little lower than the delay. This is 14 minutes.
  var timeToLive = 60 * 14;
  
  msg.shouldTurnOff = (now - msg.payload) > timeToLive;
  return msg;
  ```

#### Node 6.2: switch

This node (connected to 5.2) checks the value of the shouldTurnOff property and only passes when it's `true`.

- Name: `if shouldTurnOff`
- Property: type: `msg.` value: `shouldTurnOff`
- Rule: `==` `expression` `true`

#### Node 7.2: function

This node (connected to 6.2) creates the payload for Domoticz to set our User Variable to `0`, indicating there has no motion been detected for the last few minutes.

- Name: `Disable presence`

- Function:

  ```javascript
  msg.payload = {
      "command": "setuservariable",
      "idx": 2,
      "value": 0
  };
  return msg;
  ```

Again, make sure to replace the `idx` value with the `idx` value of your User Variable and leave `command` and `value` unchanged.

Connect this node to Node 6: *MQTT out* as well.

#### Complete flow

The complete flow should look something like this:

![1536518766005](C:\Users\Niels\Desktop\node-red-flow)

# To Do / Ideas

- ESP.deepSleep()
- Use Interrupts
- Research Node Red delay function for usage without the key/value store.
