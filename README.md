# Architecture for the Save the Bees constrained device network

## Introduction

This is the architecture for the **Save the Bees** project. It
consists of _edge cloud_ that implements the sensor network and the
gateway through which it communicates with the outside world.

It comprises the
[constrained devices](https://tools.ietf.org/html/rfc7228) and the
gateway that connects it to the outside world.

## Implementation

The implementation is done using two different technologies:

 + A very cheap and accessible design for the sensor network that uses
   the [ESP8266](https://en.wikipedia.org/wiki/ESP8266). This is the
   basic version that any beekeeper any where can assemble on a
   budget. It also the simpler to implement, since it relies in very
   well established technologies like WiFi and IPv4 accessible to
   anyone that has vere setup a domestic WiFi access point.
   
 + A more expensive version that uses
   [6LoWPAN](https://tools.ietf.org/wg/6lowpan/) and thus require a
   certain degree of familiarity with networking topics live IPv6,
   [6to4](https://en.wikipedia.org/wiki/6to4),
   [border routers](https://en.wikipedia.org/wiki/6to4), etc.
   
   
