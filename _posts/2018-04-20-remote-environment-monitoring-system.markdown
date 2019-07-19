---
layout: post
title:  "Remote Environment Monitoring System"
date:   2018-04-20 18:19:25 +0800
permalink: "/remote-environment-monitoring-system"
author: Zachary Yong
description: Remote humidity and temperature monitoring system that uses LoRaWAN technology to upload and display longitudinal environmental data.
# always put title, description, author
---

The Remote Environment Monitoring System allows users to monitor the temperature, humidity, and ambient light of a location, or multiple locations, in a convenient and maintenance-free way.

The simplest system uses two [Particle Photon](https://store.particle.io/collections/photon) devices. One of these devices is the client, which acts as the remote data collector, periodically sending the data it collects to a central hub. The other device is the server, or the central hub, which continuously listens for incoming messages from clients. The server then uploads the data for a web application to display. A more complex system can use multiple clients.

The client is fitted with a number of different [sensors](https://www.sparkfun.com/categories/23?page=all) for data collection. Here I used a [temperature and humidity sensor](https://www.sparkfun.com/products/11295) and an ambient light sensor, but any other kind of [sensor](https://www.sparkfun.com/categories/23?page=all), such as a [VOC sensor](https://www.sparkfun.com/products/14193) or a [LPG sensor](https://www.sparkfun.com/products/9405), will work. To operate remotely, the client uses a [LiPo battery](https://www.sparkfun.com/products/13851) combined with a [solar cell](https://www.sparkfun.com/products/13782) to power itself, and uses a [LoRaWAN radio transceiver](https://www.adafruit.com/product/3072) to transmit the information it collections. The [LoRaWAN radio transceiver](https://www.adafruit.com/product/3072) has an approximate range of 2 kilometers in an urban environment.

Below is a picture of the client.

![RealClient](/assets/img/rems/realclient.png)

The server, on the other hnd, is fitted with just a [radio transceiver](https://www.adafruit.com/product/3072) to receive messages from clients and send out acknowledgement messages in return.

Below is a picture of the server.

![RealServer](/assets/img/rems/realserver.png)

Every 30 minutes (or more, depending on battery charge) the client will gather humidity, temperature, and relative light data, and send the data as one concatenated `String` to the server; the server will bounce the received message to the Particle Device Cloud, while also sending an acknowledgment message back to the client; [IFTTT](https://ifttt.com/) will pick up on the new event, parse the information, and write the information to a [Google Sheets](https://www.google.com/sheets/about/) document; when the web application is opened, it will pull the organized data from the [Google Sheets](https://www.google.com/sheets/about/) document and display the information in a longitudinal and aesthetically-pleasing manner using [Google Charts](https://developers.google.com/chart/).

Below is a picture of the graphs shown on the web application after 5 days of monitoring.

![Graphs](/assets/img/rems/readings.png)
