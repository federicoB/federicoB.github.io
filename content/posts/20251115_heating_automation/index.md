+++
title = "Reverse Engineering My Heating System: Weishaupt WTC-15A Meets Home Automation "
date = "2022-11-15"
description = "A deep dive into reverse engineering my Weishaupt WTC-15A heating system and integrating it with home automation. From understanding eBus communication to building a custom Python-MQTT bridge, I share lessons learned and practical tips for DIY smart heating setups."


[taxonomies]
tags = ["home-automation","python"]
+++



A month ago, I decided to get a better thermostat for my flat, and I'm excited to share the incredible journey I've been on. It's been a fascinating DIY heating automation adventure!

It ended up being one of the most technically challenging projects I’ve ever done, even though it only took about six weeks from start to finish. So I figured I’d better write about it before I forget how crazy it got. 

I also want to help other home-automation enthusiasts avoid the same mistakes I made along the way.

**Why messing up with your Thermostat?**

It's also fun and arose from a practical need, like the best projects.

The thermostat of the existing heater in my apartment was difficult to regulate. First of all, the device had only one small display, and it was not user-friendly. We would activate the boiler, which would then heat the radiators, raising the temperature of the entire flat to the desired level. However, this process often resulted in excessive heat. 

While an excess of warmth may be preferable to a lack of warmth, our nightly sweating and subsequent sleep quality did suffer. 

When traveling for an extended period, it was necessary to deactivate the system entirely. This resulted in the thermostat losing its power and, consequently, any stored settings. Upon returning home, we would find the interior to be quite cold, and it would take considerable time to warm up again.

We lacked any degree of control over the amount of gas consumed, and we had a strong conviction that we were not operating at our optimal level. 

After considerable frustration, I have decided that it is time to explore a more efficient system. Ideally, it would be something that I could operate from my phone and that offered greater flexibility. I am aware of the existence of smart thermostats that connect to WiFi, enabling remote control of heating. They also make predictions of energy cost and try to minimize it. That would be a good solution.

**Identifying the heater and cables**

First, I had to determine the type of machine I was dealing with. That was simple; it was clearly marked on the front. 

I am in possession of a Weishaupt WTC-15A. It is a wall-hung gas condensing boiler from 2014, good, solid German quality, it can last for more than 20 years. But it came from a time of not so much interconnectivity, and it was now my duty to connect it to the world of home automation.

<img src="heater.jpg" alt="My Weishaupt WTC-15A gas condensing boiler" width="200">

Next, I identified the current thermostat. After removing it from the wall, I read "WCM-FB," which stands for "Weishaupt Condens Manager Fernbedienung" (Remote Control). Pictures on the internet confirmed this.

<img src="thermostat.jpg" alt="WCM-FB Thermostat" width="200">

At first, I was confused by the wiring. The thermostat was connected to the boiler with a blue and a brown cable, and when checking the boiler’s electrical diagram, I assumed the brown wire was a simple relay or M-Bus/Modbus line — basically, a signal cable that just turns the heating on or off. That turned out to be wrong. After researching the blue wire more carefully, I discovered that the two wires together actually form an **eBus** connection: a communication protocol commonly used by German heating system manufacturers. Unlike a basic relay that only sends on/off signals, eBus allows devices to exchange much richer data, such as temperatures, burner status, error codes, and configuration settings. In other words, it’s a small digital network that lets thermostats, controllers, and boilers “talk” intelligently over just a single **pair** of wires.

**Deciding to build rather than buy**

I began the process of researching smart thermostats that would support eBus. My objective was to identify a product within the 50-100€ range that would meet all my criteria. However, I learned that the cost would be considerably higher than I anticipated. For instance, the tado° Smart Thermostat Starter Kit V3+ is compatible with Ebus, but it has a retail price of €223.99.

Furthermore, I had concerns that the expenditure might not be worth it, as I might not be able to customize it to align with my specific requirements. Additionally, there is a possibility that I could encounter a vendor lock-in scenario, which could result in incompatibilities between the software and my future smartphone OS.

I then thought, "What is a thermostat if not a computer?" I already had a computer that was on all the time: my home server! 

Since January 2025, I have been running my home server as a NAS backup solution and GitLab hosting. The addition of a thermostat would simply be an additional service that I could operate on my home server. I simply required a method to connect my home server to the eBus Cable.  They were also close together, just one meter apart!

I knew [Home Assitant](https://www.home-assistant.io/) project already, an open-source home automation platform that lets users locally control, automate, and integrate smart devices from different brands through a single, privacy-focused system. I learned that some people have already connected their home assistant to the eBus and controlled their system. With Home Assistant i could define many types of automation for the heating. I can base them on the time of day, the temperature, and even the weather forecast. Home assistant has an advantage over coding everything yourself because the user interface is ready to use right away, and many settings can be adjusted from there.

I found a way to connect to a eBus cable. I did some research and found the [eBUS Adapter project](https://adapter.ebusd.eu/index.en.html). I bought the adapter from elecrow.com for €26.62, shipping included. 

**Installing the adapter and ebusd**

Installing the adapter was easy, the polarity of the Ebus does not matter. I installed the adapter in place of the old thermostat, which I thought I wouldn`t need anymore. Turns out I was wrong, as I’d learn later.

I additionally bought a USB Type C cable to connect the adapter to a USB port on my server.

The adapter also supports WiFi connection, but then I thought the router would be just one more point of possible failure, and since the adapter and the server are so close, better rely on a fixed connection.

Then on my server, that runs Debian, i easily installed [ebusd](https://github.com/john30/ebusd), the daemon for handling communication.

A big thank you to *John Baier* the designer and creator of the adapter and ebusd deamon. Without him i would probably never started the project at all.

Ebusd also offers a docker installation, which i prefer. But for starting i installing directly on the system without any level of virtualisation to avoid potential issues. I just wanted to try the commands.

**Reading data**

Of course, nothing goes completely smoothly. On Linux, I encountered a permissions issue where my user couldn't access `/dev/ttyACM0`. The device file was created with the `root:dialout` group. I added my user to the dialout group and verified it with `ls -l /dev/ttyACM0`. Fixed! Back in business!

Upon launching ebusd and scanning, I found something odd: the bus scan detected my unit as a Kromschroeder “SC” (solar system) rather than recognizing the Weishaupt WTC-15 properly. This is because ebusd, by default, uses certain configuration files (CSV definitions) to interpret the bus nodes. Without the correct config for a specific model, you’ll get mis-recognition or incomplete data.

However, I found the Weishaupt configurations on GitHub.: [J0EK3R/ebusd-configuration-weishaupt](J0EK3R/ebusd-configuration-weishaupt). This included the CSV files and definitions for numerous Weishaupt units. I cloned the repository locally and configured ebusd to use the J0EK3R repository as the configuration source. Using the correct configuration meant that ebusd loaded the correct definitions and could interpret the eBUS telegrams from my heater in the correct context (addresses, registers, units, endianness, etc.). J0EKR3 struggled as well to determine this configuration. You can read about his experience in this [forum](https://forum.fhem.de/index.php/topic,61017.0.html). 

Now, in the logs, I could see "BrennerAus," as well as other numbers that I still didn't understand. 

**Reverse-engineering commands**

Before I could control the heater from Home Assistant, I wanted to check if I could do it from the command line. I used the definition from J0EK3R of the [f6 address](https://github.com/J0EK3R/ebusd-configuration-weishaupt/blob/master/weishaupt/Documentation/f6.md) which is the boiler controller, and I tried to guess the hex code, but I only got errors. When there were no errors the boiler still did nothing. At this point, I didn't know whether the J0EK3R configuration was wrong or whether the hex I had determined was incorrect. I reached a point where I was ready to give up. After all, I never thought I would go so low-level and write bytes directly into a bus. I thought that with the ebusd adapter, everything would work straight away and I would be able to control the heating with Home Assistant. Instead, it seemed I had to control everything myself, but I did not know why. I didn't know the correct syntax for the command.

But how could I learn it? From whom or what? The answer was my old thermostat.

Ultimately, Ebus is a bus to which multiple devices can be connected, including masters and slaves. I thought I could connect everything together — the boiler, the old thermostat and the Ebus adapter — and use the adapter to listen to the communication between the thermostat and the boiler, and learn from it. 

<img src="reverse_edited.jpg" alt="My reverse engineering setup" width="800">

Here’s a zoomed-out picture showing how professionally I was working:
<img src="pile.jpg" alt="My reverse engineering setup" width="300">

I had never connected cables in that way before and I had no idea if it would work, but it did! I could read the traffic on the bus and see the hex. Then, by adjusting the controls on the old thermostat, I could see how the desired temperatures were encoded and determine the endianness. I used ChatGPT to decode the hex values, giving it the hex message and the assumed structure from J0EK3R as input.

I'll give you an example of how deep I had to go to decode it. For example, I could see my thermostat was transmitting the message `30f1050709aa032d020080ffffff` and this is how I decode it:

<img src="decoding.jpg" alt="Decoding of ebus message" width="800">

But I also saw that the boiler would not stop when it reached the desired temperature. However, it was also transmitting the boiler temperature, and the thermostat would send a "turn off" message when reached. My code would have to do the same.

I also noticed that I need to continuously instruct the boiler to stay off; otherwise, it turns on by itself when it detects that it is cold outside.

By mimicking the same messages that the Weishaupt thermostat was sending, at least I didn’t have to worry anymore about breaking the heating, as I was sending manufacturer-approved commands!

**Integrating with MQTT and Home Assistant**

Although I could now control my heater using the command line, it was definitely not the most user-friendly way. Next, I wanted to connect it to Home Assistant! I found [an integration](https://www.home-assistant.io/integrations/ebusd/) for ebusd but it is generally recommended to use an [MQTT broker](https://en.wikipedia.org/wiki/MQTT) between ebusd and homeasistant for the following reasons:

* Using MQTT decouples Home Assistant from the eBus protocol, reducing slowdowns, timeouts, and retries compared to direct eBusd integration.
* Last known values can persist even if HA restarts, the state can be retained.

For these reasons, John developed an [MQTT integration for ebusd](https://github.com/john30/ebusd/wiki/MQTT-integration). It has an interesting self-discovery feature. At startup, messages are sent to the MQTT topic /homeassistant/ describing the controllable devices and how they can be controlled. This defines the control topics and the expected payload syntax. The payload definition is determined from the .csv files of the ebus telegram structure. 

I just needed an MQTT broker to run as a service. I chose [Mosquitto](https://hub.docker.com/_/eclipse-mosquitto) because it was the first one I found, was easy to install, and was highly recommended by everyone. I haven't had any problems with it, and I'm happy.

As soon as I fed into the ebusd startup command, the [mqtt-hassio.cfg](https://github.com/john30/ebusd/blob/master/contrib/etc/ebusd/mqtt-hassio.cfg) and my Mosquitto service name started to appear in HomeAssitant several devices and entities. 
This confused me because I only had one device, my boiler, but instead I was getting ten devices. However, one of them was showing the boiler temperature, the outside temperature (the boiler is connected to an outdoor thermometer), and the burner phase (on/off), for example. However, I was also getting a lot of empty values, and there was no way to control the boiler from Home Assistant; I could only read the values.

The culprit could be the mqtt-hassio.cfg configuration or the .csv files from J0EK3R, so I started investigating both. 

I won't go into too much detail, but it was difficult to understand both, and I couldn't figure it out. Even after understanding how the CSV files are structured, I could not figure out how they influence the discovery payload through the MQTT-Hassio.cfg file. I knew the discovery message I wanted, but I couldn't generate it.

Moreover, a disclaimer from the J0EK3R repository states, "Maintainer wanted: I no longer have a Weishaupt burner, and I don't have any spare time for this project." I also did not know who to ask for help. John also seemed busy and inactive on GitHub, so I had to figure this out by myself.

I learned about the newer `.tsp` [format (“Type-Specification”)](https://github.com/john30/ebus-typespec)  used by the config repo. It is more readable and supports writing definitions elegantly. Initially, I easily converted the .csv config to the .tsp format with ChatGPT, and then I learned the language's documentation to make some adjustments. I had to figure out where to put the .tsp file and how to name it. You need to run a .js script that converts the .tsp back to .csv so ebusd can read it. I thought this new approach and format would help me create writable entities. It didn't. At least my configuration was more readable now, and I could share it on GitHub. I made a  [Pull Request](https://github.com/john30/ebusd-configuration/pull/532) for it.

Every hour, I was diving deeper and deeper. I read all the GitHub issues regarding Weishaupt. I tried multiple obscure ebusd parameters, but it still wasn't generating the correct discovery payload with writable entities in Home Assistant.

To understand why, I decided to debug the C++ code of ebusd and follow the execution of generating the discovery payload. 

Disclaimer: I can understand C++, and I have written C++ in the past, but it is definitely NOT my favorite language.

Nevertheless, I configured a `.devcontainer.json` that was missing, so I can have all the dependencies needed to debug `ebusd` without needing to install it on my host machine. I also made a [pull request](https://github.com/john30/ebusd/pull/1604) for that.

I followed the code execution and tried to understand it. Honestly, it was not easy, and I still did not understand how `mqtt-hassio.cfg` works.

Additionally, I do not like in C++ how a parameter is passed to a function, modified internally by the function, and this is the "result" of the function instead of being declared as a return value of the function, as in Python.

In the end, I found the culprit in the code: a comment stating:
``````
// multi-field message is not writable when publishing by field or combining multiple fields in one definition, so skip it
``````

My message to control the boiler was indeed a multi-field message. If I declared it as writable, ebusd would simply skip it.

I assume this is because ebusd does not implement a memory of the latest value set for a field in a multi-field message, so it cannot progressively compose a multi-field message from parts of it.

**Building My Own Python Bridge**

I attempted to modify the ebusd code for a while, but it was far too complicated. Ultimately, I decided that the MQTT integration was not sufficient for my needs, and I could instead write my own solution. My plan was to create a service, written in Python, that would bridge ebusd and MQTT. This service would interact with ebusd using `ebusctl` and utilize the [paho-mqtt](https://pypi.org/project/paho-mqtt/) library to read and write messages to the broker.

This is the architecture scheme that shows the setup i was aiming for:

<img src="architecture.jpg" alt="Architecture drawing showing ebusd-ebusmq-mosquitto together" width="800">

I began by crafting the discovery payload I had been aiming for in MQTT, which immediately resulted in writable entities in Home Assistant. Next, I implemented functionality to listen for command messages from Home Assistant and trigger the corresponding commands on the boiler. Following that, I developed a loop to continuously send the desired status, ensuring the boiler remained on or off as intended.

Afterward, I focused on reading values from `ebusctl` to monitor the actual boiler temperature and the outside temperature. These values were first read into `ebusmq` and then sent to Home Assistant via MQTT. Finally, I packaged everything into a clean container and set up a GitLab CI/CD pipeline for deployment on my home server.

This process took me slightly less than three days and brought me immense satisfaction, as I was no longer stuck debugging and could instead create my solution.

I haven’t released `ebusmq` on GitHub yet because, so far, it’s very tightly coupled to my specific boiler model. My goal is to enhance it so that it can parse a `.tsp` file and dynamically generate an internal Python data model for the eBus telegrams.

**Home Assistant Automation**

To monitor the temperature in each room of the flat, I purchased three SONOFF SNZB-02D Zigbee Temperature and Humidity Sensors. They are all connected to a Zigbee USB dongle installed on the server, and their data is stored in Home Assistant. I calculate the average flat temperature, and the triggering of the heating system is based on that.

My basic but very effective automation heats the boiler to 50 degrees and then stops. It does not warm up again for another hour. At night, the heating system stays off. All these parameters are adjustable via Home Assistant. I recently experienced the convenience of a warm house after returning from my trip to Hamburg. While on the train, I was able to trigger the heating system remotely!

**Key Takeaways:**

* **Config files matter**: eBUS isn’t plug-and-play; you need correct configuration files. Without the proper CSV/TSP definitions, your system will misinterpret data. You’re lucky if you can find community configurations, but you might need to do some research or reverse engineering for your device.
* **Reading is easier than writing**: Many home automation enthusiasts stop at getting a nice dashboard of current values in Home Assistant and don't progress to controlling their devices from there.
* **Know When to Build**: Give yourself a maximum time limit for debugging an existing tool. When that limit is exceeded, write your own solution.

**Potential Next Steps**

* **Connect to weather forecasts**: Avoid warming the flat excessively when a sunny day is expected or start heating at night if very cold mornings are predicted.
* **Build nice dashboards in Home Assistant**: For example, show flow temperature, return temperature, on/off status, history graphs, and efficiency metrics.
* **Generate alerts**: For example, alert when the pump hasn’t been running for X minutes or when an error code is detected in the heating system.
* **Reinforcement learning**: Enable the system to learn from temperature patterns, occupancy, and energy usage to optimize heating schedules automatically.

