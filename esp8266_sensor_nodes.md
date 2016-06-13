# Cheap high-performance sensor nodes using the ESP8266

## Introduction

The [ESP8266](https://en.wikipedia.org/wiki/ESP8266) is very useful
device for developing **cheap** sensor nodes architectures. It boasts
a very low price point coupled with a large community and multiple
SDKs available. 

 + [Lua SDK](https://github.com/nodemcu/nodemcu-firmware/): It uses a
   _custom_ Lua 5.1 implementation taken largely from the
   [eLua](http://www.eluaproject.net/) project.
   
 + [FreeRTOS SDK](https://github.com/espressif/ESP8266_RTOS_SDK): It
   offers the possibility of more _close to the metal_ development
   coupled with a real time operating system possibilities. 

 + [Arduino SDK](https://github.com/esp8266/Arduino): this is a pure
   community driven effort. It taps into the Arduino ecosystem and
   large community.
   
## Discussion
 
In IoT the boundary between server and client is tenuous to say the
least. In fact IoT networks of
[constrained devices](https://tools.ietf.org/html/rfc7228) are much
more of a peer-to-peer (p2p) network than a _classical_ server-client
model as seen on the web. As such it poses new problems and requires
new ways of thinking about network architecture, protocols and
interfaces. The same applies to the application protocol layer. 

This realization has prompted a flurry of acitvity from the
[IETF](http://ietf.org) to define new standards that are open and
obtained through a consensus process in the Internet community at
large instead of being captured by special interests that promote a
particular agenda.

Here are some of the working and research groups from the IETF
addressing the challenges posed by IoT:

 + [Constrained Restful Environments](https://tools.ietf.org/wg/core)
 + [6LoWPAN](https://tools.ietf.org/wg/6lowpan/)
 + [6lo](https://tools.ietf.org/wg/6lo/)
 + [CBOR Object Signing and Encryption](https://tools.ietf.org/wg/cose/)
 + [Authorization in Constrained Environments](https://tools.ietf.org/wg/ace/)
 + [Lightweight Implementation Guidance](https://tools.ietf.org/wg/lwig/)
 + [Thing 2 Thing Research Group](https://datatracker.ietf.org/rg/t2trg/)
   
Taking in consideration the above ongoing work we propose a modern
architecture for cheap sensor nodes based on the ESP8266.


## Architecture

We'll make use of all the available features on the ESP8266 to save as
much energy as possible and have separation of concerns, while at the
same time be effective and simple.

## Energy saving measures

### Deep sleep

The WiFi is the main driver of power consumption, we therefore have
all interest in using it as little as possible.

Using the
[rtcfifo](http://nodemcu.readthedocs.io/en/master/en/modules/rtcfifo/)
is possible to wake up periodically for reading sensor data, collect
data into the RTC memory slots and then wake up to send the data in
batch. This allows decoupling the wake up for sending data from the
wake up for reading sensor data.

As an example let's consider a sensor reading frequency of **every 5
minutes** and a publishing frequency of **once every hour**.

From the
[rtctime](http://nodemcu.readthedocs.io/en/master/en/modules/rtctime/)
module:

```lua
-- Don't wake up the radio. Sleep for 5 minutes and wake up to read
-- the sensors. Reading the sensor takes one second max.
local minute_usec = 60000000
rtctime.dsleep_aligned(5 * minute_usec, 1000000, 4)

-- Sleep for 59 minutes and be awake for 1 minute pushing the data
-- out. 
rtctime.dsleep_aligned(59 * minute_usec, minute_usec, 1)
```

### Using fixed IPs

When the radio wakes up the best option is to use a **fixed** IP since
it avois having to wait for the DHCP server from the access point to
grant a lease on a non fixed IP. Thus we can save more energy and
reduce the period when we need to be with radio in full power.

## Keeping accurate time

The ESP8266 doesn't have a Real Time Clock (RTC). From the 
[documentation](http://nodemcu.readthedocs.io/en/master/en/modules/rtctime/#rtc-time-module):

<blockquote> 
Time keeping on the ESP8266 is technically quite challenging. Despite
being named RTC, the RTC is not really a Real Time Clock in the normal
sense of the word. While it does keep a counter ticking while the
module is sleeping, the accuracy with which it does so is highly
dependent on the temperature of the chip. Said temperature changes
significantly between when the chip is running and when it is
sleeping, meaning that any calibration performed while the chip is
active becomes useless mere moments after the chip has gone to
sleep. As such, calibration values need to be deduced across sleep
cycles in order to enable accurate time keeping. This is one of the
things this module does.
</blockquote>

So in order to have a high-quality reliable clock we have to use
[SNTP](https://en.wikipedia.org/wiki/Network_Time_Protocol#SNTP)
Network Time Protocol. This relies on the
[sntp](http://nodemcu.readthedocs.io/en/master/en/modules/sntp/)
module. This module works in cooperation with the
[rtctime](http://nodemcu.readthedocs.io/en/master/en/modules/rtctime/)
module.

```lua
-- Synch up the time from an NTP server.
-- One in the de.pool.ntp.org pool
-- IP addresses: 212.18.3.18, 144.76.207.204
-- 129.70.132.35 and 109.75.223.1
sntp.synch(
  '129.70.132.75',
  function(sec, usec, server)
    print(string.format('Synch NTP time with %s: %s.%s',
                         server,
                         sec,
                         usec))
  end, 
  function()
    print('Cannot synch time via NTP'.)
  end
)

-- Now we get accurate time.
print(string.format('UNIX Epoch timestamp: %s.%s', rtctime.get()))

```

## Service and node discovery

There are two main options for implementing this functionality.

 + [Multicast DNS](http://www.multicastdns.org/).
 
 + [CoRE Resource Directory](https://datatracker.ietf.org/doc/draft-ietf-core-resource-directory/)

### Easy identification of a node: multicast DNS (mDNS)

[Multicast DNS](http://www.multicastdns.org/) was developed for
networks where no DNS server exists. This is particularly useful for
networks of constrained devices where we aim for the simples possible
solution.

The [mdns](http://nodemcu.readthedocs.io/en/master/en/modules/mdns/)
module allows to register or de-register a node in a mDNS enabled
network. This simplifies discovery and also makes it possible to
assign unique IDs to devices that **survive** accross network
changes. Imagine a node being in network A and then being moved to
network B. It will keep the name among networks.

As the name implies multicast DNS requires multicasting. Such is not
usually allowed in shared infrastructure, but it is usually allowed in
LANs that are personal or with a delimited perimeter.

```lua
-- When the node powers up, register it with
-- mDNS service.
mdns.register('foobar.local',
              { service = 'coap', 
                port = 5683,
                description = 'Foobar node',
                location = 'somewhere',
                proto = 'udp'})
```

Note that this has many limitations since it will show up the service
as `_coap.tcp.local` which is incorrect. Since CoAP here works over
UDP and not TCP. But that's a limitation of the current library.

```C
/* nodemc_mdns.c file */
mdns_set_servicename(const char *name) {
	char tmpBuf[128];
	os_sprintf(tmpBuf, "_%s._tcp.local", name);
	if (service_name_with_suffix) {
	  os_free(service_name_with_suffix);
	}
	service_name_with_suffix = c_strdup(tmpBuf);
}
```

### Using CoRE Resource Directory

Alternatively to mDNS we can use the
[CoRE Resource Directory](https://datatracker.ietf.org/doc/draft-ietf-core-resource-directory/)
to have a more compreensive way of defining resources that doesn't
rely on multicasting. This approach is preferred in terms of modern
more widely deployable architecture.


### CoAP as the application protocol

[CoAP](https://tools.ietf.org/html/rfc7252) Constrained Application
Protocol is the application protocol being used.

Node MCU offers a not yet
[complete](http://nodemcu.readthedocs.io/en/master/en/modules/coap/)
implementation.

 + The client implementation lacks a way to specify headers.
 + The server lacks support for PUT and DELETE. 
 + No support for [CBOR](http://tools.ietf.org/html/rfc7049) Concise
   Binary Object Representation.
   
   
### SenML as the output format

[SenML](https://tools.ietf.org/html/draft-jennings-core-senml-06)
Sensor Markup Language is the output format to be used. To that end we
need to use the
[CJSON](http://nodemcu.readthedocs.io/en/master/en/modules/cjson/)
module from NodeMCU.

Example: 

```javascript
// Array with a single SenML record.
[{ "n": "urn:dev:ow:10e2073a01080063", "v":23.1, "u":"Cel" }]

// Array with two SenML records. Notice the use of timestamps
// to distinguish between (possibly) consecutive readings.
[
    { "n": "urn:dev:ow:10e2073a01080063",
      "t": 1276020076, "v":23.5, "u":"Cel" },
    { "n": "urn:dev:ow:10e2073a01080063",
      "t": 1276020091, "v":23.6, "u":"Cel" }
]

```

For reference:

 + [List of attributes](https://tools.ietf.org/html/draft-jennings-core-senml-06#section-6) 
 + [List of physical units](https://tools.ietf.org/html/draft-jennings-core-senml-06#section-12.1)
 + [SenML MIME media type](https://tools.ietf.org/html/draft-jennings-core-senml-06#section-12.2)
 
 The MIME media is defined as `senml+json` for JSON SenML
 payloads. 

## Sensor node API 

It's implemented using both a CoAP client and a CoAP server.

### CoAP client

#### Data publishing

Publishes data to a CoAP-HTTP proxy reading from the RTCFIFO described
above. This requires radio wake up. 

#### Resource discovery registry

See
[CoRE Resource Directory](https://datatracker.ietf.org/doc/draft-ietf-core-resource-directory/)
for examples. To be elaborated.

### CoAP server

Implements a series of endpoints. 

#### Updating the sensor reading frequency

 * request

```shell
POST /rf
``` 
with a body:

```javascript
{
  "read": <number>,
  "unit": <unit>
}
```
where `<number>` is an number (float or integer) and `<unit>` is one of:

 + "m": minutes 
 + "s": seconds
 
Note that due to delta compression being used in timestamps the
frequency has to be greater or equal than 8.5 minutes. Hence the
maximum value is:

```javascript
{
  "read": 8.5,
  "unit": "m"
}
```
See the
[rtcfifo](http://nodemcu.readthedocs.io/en/master/en/modules/rtcfifo/#rtc-fifo-module) documentation. 
 
 * response
 
```shell
2.01 Created
```

#### Get the sensor reading frequency

 * request

```shell
GET /rf
``` 
 * response

```shell
2.05 Content
```

```javascript
{
  "read": 5,
  "unit": "m"
}
```

Sensors are read every 5 minutes.
 
#### Updating the sensor publishing frequency

 * request

```shell
POST /pf
``` 
with a body:

```javascript
{
  "read": <number>,
  "unit": <unit>
}
```
where `<number>` is an number (float or integer) and `<unit>` is one of:

 + "m": minutes 
 + "s": seconds
 + "h": hours

Example publish data every hour:

```javascript
{
  "publish": 1,
  "unit": "h"
}
```
See the
[rtcfifo](http://nodemcu.readthedocs.io/en/master/en/modules/rtcfifo/#rtc-fifo-module) documentation. 
 
 * response
 
```shell
2.01 Created
```

#### Get the sensor publishing frequency

 * request

```shell
GET /of
``` 
 * response

```shell
2.05 Content
```

```javascript
{
  "read": 0.5,
  "unit": "h"
}
```

Publishes every 30 minutes (1/2 hour).

## Closing remarks

Note that the requests to the CoAP server running on the node and the
synchronisation of the time using NTP can only be done whenever the
radio wakes up. Since it requires network connectivity.

## TODO

 + Use [swagger](http://swagger.io) for the API definition.
