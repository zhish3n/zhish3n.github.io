---
layout: post
title:  "Documentation: Mock Garage and Remote Control"
date:   2018-02-05 18:19:25 +0800
permalink: "/docgaragecontrol"
description: Combining Particle API's IOT functionality with Ionic and Cordova. Implementation of Pub/Sub techniques to control a garage door with multiple clients.
# permalink: /:categories/:year/:month
---

In this project I use a Particle Photon to build a mock garage that can be controlled by a web app and a mobile app. Any input or output on either the garage, web app, or mobile app is synchronized across all three platforms.

The garage has a number of requirements:

* Includes a (conceptual) door that
  * can be toggled (on or off) to automatically start the closing process after a specified time has elapsed (default 5 seconds).
  * can be in any one of the following states:
    * Closed
    * Opening
    * Open
    * Closing
    * Halted
    * Faulted
* Includes a button that, when pressed,
  * if the door is closed, start opening the door.
  * if the door is opening, halt the opening process.
  * if the door is halted after an opening process, start closing the door.
  * if the door is open, start closing the door.
  * if the door is closing, halt the closing process.
  * if the door is halted after and closing process, start opening the door.
* Includes a garage light that
  * is on when the door is in a moving process.
  * when the door stops moving, fades to off over 5 seconds after a specified time has elapsed (default 5 seconds).
* Includes a light that is on when the door is opening and off when it is not.
* Includes a light that is on when the door is closing and off when it is not.
* Includes wires acting as "sensors" that will toggle
  * the door going from an opening state to to an open state.
  * the door going from a closing state to a closed state.
  * the door going from either an opening or closing state to a halted state.
  * the door going from either an opening or closing state to a faulted state.

The first step is building the mock garage using the Particle Photon. The schematic is shown below.

![ServerSchematic](/assets/img/garagecontrol/server_fritzing.png)

With the mock garage set up, the next step is to program the functionality. First, we need to set up the hardware.

```c
// INPUT CONSTANTS
const int doorButton = D0;
const int closedSwitch = D1;
const int openSwitch = D2;
const int faultSignal = D3;
const int doorOpening = D4;
const int doorClosing = D5;
const int garageLight = A5;

void setup() {
  setupHardware();
}

void setupHardware() {
  Serial.begin(9600);
  // INPUTS
  pinMode(doorButton, INPUT_PULLUP); // door button
  pinMode(closedSwitch, INPUT_PULLUP); // closed switch
  pinMode(openSwitch, INPUT_PULLUP); // open switch
  pinMode(faultSignal, INPUT_PULLUP); // fault signal
  // OUTPUTS
  pinMode(doorOpening, OUTPUT); // door opening
  pinMode(doorClosing, OUTPUT); // door closing
  pinMode(garageLight, OUTPUT); // garage light
}
```

Particle's API makes it relatively straightforward to interact with IOT projects through simple web applications built with HTML, CSS, and JS. Using Apache Cordova, we can serve these web applications as hybrid mobile applications on various devices without needing to do much work. However, mobile applications built on Apache Cordova are relatively slow, cannot access a number of native features, and do not follow the "look and feel" of the operating system it runs on (it depends on your own CSS styling). One option that improves on these points is the Ionic Framework.

Getting started with the Ionic Framework is relatively straightforward as there is a lot of documentation on its UI components. But, since Ionic uses Angular, using the Particle API to connect your mobile application to your IOT project is more difficult. Prior to this project, I built a mini garage door using a Particle Photon, which could be controlled by a remote control web application running Particle's API. In this project, I use the Ionic Framework to build a remote control which also uses Particle's API, which means that the garage door, the web application remote control, and the mobile application remote control are all synchronized.

I set up a new Ionic project with Ionic CLI, called myGarage. This is as simple as opening up a command prompt and typing `ionic start myGarage blank`. Note that `blank` is so because we don't want to use any of Ionic's starter templates.

After the project has been made, observe the project structure. Ionic has pretty good documentation on project structure (link here), but basically all the work you'll be doing will be in the /src folder. The first thing we want to do is open up `/src/index.html`. This is the main entry point for the app, though its purpose is to set up scripts, CSS includes, and bootstrap, or start running our app. We won’t spend much of our time in this file, but we do want to add our particle min js file here.

app.module.ts is the entry point of the app, and here we set the root to MyApp, in app.component.ts. This is the first component that gets loaded in our app, and in app.component.ts, we're setting our template to src/app/app.html, which will load our rootPage, which is our HomePage.

---------------

The remote control web app includes two JS files. The first file, `GarageApp.js`, includes all the functions needed to navigate between pages in the web app. An example is shown below.

```js

```






If we go to src/pages/home, you can see the files that make up this page. The files we are interested in are home.html and home.ts. The first file, home.html, is where we design the look of the page. Again, Ionic has plenty of documentation on UI, so it was relatively straightforward for me to throw together a simple login page, shown below.

The purpose of this login page is to make sure that we are connected to the Particle Device Cloud.
