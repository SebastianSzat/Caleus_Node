# Caelus Node (CNode)
### A distributed environmental monitoring system, built one stage at a time.

---

## How it started

I moved to Auckland in 2024 on a working holiday visa. The plan was to stay. Finding work as a foreigner in that category turned out to be harder than I expected. The market was tight, the trust in foreigners were low, and I spent a good few months just getting my bearings, trying to find the correct angles. I prepared a lot before the move, but still felt like I am running in circles without result. Early 2025 another Hungarian who'd made the same move a few years earlier, helped me figure out how things worked and started pointing me toward meetups worth going to.

I have eight years of database work behind me, more on the development side than pure admin. The details are on my LinkedIn profile, and in the stories on my other projects in this repo. I settled in the Database field because that was a good choice at the time, and I got help with it, but I was always interested in the hardware field too. I like to tinker, I like to build, I had a few interesting projects, like a drink tap created in an osb box, cooled with Peltier components, and distributed with vacuum pumps running on a 12V circuit with fan cooled parts.

From April 2025 I was going to meetups regularly. In June I went to NZ Hardware meetup for the first time. It was a hardware-focused event at Outset, a co-working venue in the Balfour Road area. People stood up and showed things they'd been building, to a room that was part audience, part companies looking for people, part people just wanting to be around other builders. I went three times total. I tried to pitch myself, and I pulled it through, althoug with database only knowledge I did not get traction on a hardware centered audience.

I wanted to show I can think in hardware too, so I started to think about a project. New Zealand has a real UV problem, the ozone is thinner there than what I am used to in Europe, and an afternoon that feels fine can turn on you faster than you'd think, especially for kids. Auckland weather also moves fast and it's local, wind and rain and UV all varying within a short distance. A school trying to decide whether to keep kids inside for lunch doesn't have much to work with unless it's a large city installation. Same for a park running an event. Either expensive infrastructure or nothing.

I started sketching out what a small, cheap, solar-powered sensor node might look like. Something you could put up in enough numbers to actually cover a neighbourhood. Something that routes data and alerts to whoever needs them without requiring a smart city budget or a dedicated IT team. I named it Caelus Node, CNode for short. Caelus is the Roman personification of the sky... weather, the vault of heaven, all of that. It fit.

---

## The document that sat on my laptop

Before the idea went anywhere I turned it into a High-Level Design document with some help of one of the LLM-s to bridge the gap in my experience with hardware.
I was going to present this at the meetup. Then a job interview came up that needed proper preparation and the meetup went on the back burner. The document sat on my laptop for months without being opened.

__The "document" or the idea summarized:__
Three product tiers. A basic unit around $50 covering temperature, humidity, rainfall, UV and noise with one communication method. An enhanced tier around $59 adding wind sensing and BLE. A professional tier at $82.50 with full modularity, GPS, vibration, redundant sensors on the critical channels, and LoRa plus BLE plus NB-IoT combined. I knew those numbers would shift as soon as I touched real components (they always do) but the point was to show I'd actually looked at the parts and understood the cost structure wasn't crazy.

Connectivity centred on LoRaWAN, one gateway covering multiple nodes per square kilometre, BLE for offline mobile sync, NB-IoT as a fallback for remote spots.

On data access I drew a line I wasn't willing to move: anything safety-critical (destructive wind, extreme UV, significant vibration) goes out to everyone free, no subscription required. The subscription model covers real-time data and detailed analysis, not warnings.

The part of the document I kept coming back to was a simple idea, visual indicator panels near school entrances and park gates. No app, no phone, just a light.

Schools: blue for normal, yellow when sun protection is worth thinking about, orange for high wind, red when staying inside makes sense.

Parks: blue for good conditions, yellow for moderate heat or UV, orange when it's windy or loud enough to be careful around trees, red for a temporary hazard, flashing red for storm conditions.

Someone who don't have an app can read a coloured light on a door or post. A parent dropping off kids glances at the gate. No screen required.


---

## Why I started building it

I came back from Auckland. I'd like to go back - it got into my head the way some places do.

The idea kept coming back too. Then early 2026 the AI tooling conversation started getting loud, and I did a fairly honest look at where my skills were and where they weren't. Eight years of database work is real and I'm good at it, but the It world seems like it will shrink, competition will be fiercer, and I need an edge to be able to thrive. Iworked on data from many angles, but never on the hardware and firmware side. I had the HLD sitting there already. It seemed like a waste not to try.

So I started.

I used Claude as a working tool throughout, understanding Node-RED flows, debugging dashboard schema problems, talking through sensor choices, working out the alert routing logic... and most of all, helping me get into the field.

---

## Software before hardware

I built the whole data pipeline in Node-RED with mock data before ordering anything.

Ten inject nodes simulating sensors (temperature, wind speed, wind direction, UV index, rainfall, noise level, battery percentage, vibration, humidity, air pressure). A Join node collects them into one object. A timestamp function adds an ISO timestamp and station ID. From there the data goes three ways: dashboard, alert router, debug output.

The alert router feeds four switch nodes, one each for UV, wind, noise and vibration, with thresholds at each tier. UV moderate fires a park and school alert. UV extreme fires a global severe alert. Noise above 65dB sustained or 85dB as a spike routes to a police report topic. Vibration above 8 m/s2 goes out as an earthquake-level alert. Each branch builds a JSON packet and publishes via MQTT,`caelus/alerts/park`, `caelus/alerts/police`, `caelus/alerts/severe` and so on.

A second tab subscribes to those topics and feeds a FlowFuse Dashboard 2.0 interface, gauges, alert log, history charts, popup notifications.

Dashboard 2.0 has different node schemas from the deprecated package, that I found some guide to, so it took longer than expected to get right. Orphaned config nodes surviving tab deletion, MQTT client ID conflicts causing duplicate messages, gauge properties failing without any obvious error. A series of problems, each one hidden behind the previous one.

The Node-RED mock flow is the first commit in this repository. It's rougher than I'd like and parts of it will need rethinking when real sensor data comes in. But it works, and it was a starting point.

---

## Hardware

Components came from multiple places -- Botland, Reichelt, Conrad, and a local hardware stores. Budget under 100 euros for a working first prototype... or at least I thought. Buying only one part is always more expensive, instead of 3-5$, it can cost up to 10, and shipping especially, if you can't find everything in one place is a huge extra cost, sometimes more than the part itself. So including a new soldering iron, flux, soldering wire, testboard..... etc... it was around the 200$ or more. I really hope I will be able to go back to under 100$with a finished product, but I may need to invest in a 3d printer at some point, if I want to make this modular, and not spend a ton extra on paying others to model and print for me.

Main parts (I will add the full list with photos later):
| Sensor | Chip | What it covers |
|--------|------|----------------|
| Temperature, humidity, pressure | BME280 | Three readings from one chip |
| UV | GRV UV-A/B analogue | UV-A and UV-B |
| Noise | MAX9814 microphone amplifier | Analogue dB estimation |
| Vibration | MPU-6050 on GY-521 board | 3-axis accelerometer and gyroscope |

Controller: ESP32-DevKitC-32E (dual-core, 240MHz, WiFi built in, enough GPIO for all sensors).

Power: two 18650 cells in a replaceable holder, charged by a TP4056 with protection circuit from a 1W 6V solar panel. IP65 junction box on a fence post.

Wind and rainfall sensors are deferred. Sadly I could not buy them as parts, i could buy a full metheorological station alone, and canibalyze it, or buy industry level parts that would blow my budget. I will build them myself later, it will be a good learning too.

---

## A note on what isn't documented here

Everything up to this point happened across too many places - conversations, debug sessions, browser tabs, different shopping carts, and under shower thinking - to reconstruct neatly into git commits. This story is the record of that phase.

From here each stage gets committed as it happens. First commit is the Node-RED mock flow. Everything after is documented as it actually occurs, although probably not in a too fine-grained way.

---

## Where things stand

- [x] High-Level Design document (Auckland, June 2025)
- [x] Node-RED pipeline with mock data(March 2026)
- [x] FlowFuse Dashboard 2.0 - gauges, charts, alert log
- [x] Mosquitto MQTT broker running locally
- [x] Alert routing to park, school, police, global severe
- [x] Receiver dashboard with per-channel alert logs
- [x] Hardware ordered and delivered
- [x] Soldering kit, breadboard, jumper wires, lab board, flux...
- [ ] Breadboard prototype - sensors tested individually
- [ ] ESP32 firmware (Arduino IDE / C++)
- [ ] Real sensor data replacing mock inject nodes
- [ ] Final assembly in IP65 enclosure
- [ ] Wind and rainfall sensors - DIY phase
- [ ] Multi-node planning
- [ ] Visual alert light integration

---

## Why this is public

I'm putting this here partly as a portfolio piece, partly as a record for myself. Looking back at how something started is useful when you're stuck in the middle of it.
All of this is available to be used, except the name, and the idea. Please let me try to finish it myself, and don't beat me to the punch, and monetize it.

The hope is that this becomes something finished, a working node, maybe eventually a small product or an open contribution to something. Not certain about the destination but fairly committed to the journey.

If you're hiring for someone who understands data pipelines, can work through a system from concept to component level, and is honest about what they're still learning, this repository is a reasonable place to look.

Production code and final hardware specs will stay private if this gets to a commercial stage. Everything here is the real process.

---

# Story mode

### All of the parts arrived this week (26-03-2026)

https://github.com/SebastianSzat/Caleus_Node/blob/main/Hardware/First_mockup_parts_arrived.JPG
![First mockup parts arrived](/Hardware/First_mockup_parts_arrived.JPG?raw=true "First mockup parts arrived")

Next will be the software environment on the laptop when I have the time for it, testing the sensors one by one, and building the first "brain".

---
## Getting the ESP32 Working (30-03-2026)
## Arduino IDE setup and first upload

---

## Installing the tools

Download Arduino IDE 2.x from arduino.cc. The install is straightforward, next-next-finish.

ESP32 is not in Arduino by default so you add it manually. Open **File → Preferences**, paste this URL into the Additional Boards Manager URLs field:

```
https://raw.githubusercontent.com/espressif/arduino-esp32/gh-pages/package_esp32_index.json
```

Then **Tools → Board → Boards Manager**, search `esp32`, install **esp32 by Espressif Systems**. It downloads around 300MB and takes a few minutes.

Once done, go to **Tools → Board → esp32** and select **ESP32 Dev Module**.

---

## I am lucky I only thrown out 95% of my old cables

Plugged the board in. Red light on the ESP32, it's getting power. Opened Arduino IDE, went to Tools → Port. Greyed out. Nothing.

Opened Device Manager. No Ports category at all. Tried to find in other categories, and still nothing. 

The board uses a CP2102N USB-to-serial chip (I had to use a bright fleshlight and a correct angle to be able to read it... I may need to buy a magnifier), so I downloaded the Silicon Labs driver and installed it. Still nothing. I installed a second driver variant... still nothing... Ran the universal driver installer and manually added registry entries... still nothing.

Tried 3 different micro-USB cables... still nothing.

The fourth cable worked, and I have an extra fifth piece that I did not try, and will not try until I lose the lucky number four. I thought I have only one of these lieing around, as only my old BT headphones are using them... I feel lucky to be wrong. It's not expensive, but a hussle to get one just to start.

Most micro-USB cables floating around in drawers are charge-only, they carry power but the data wires inside are either missing or just not connected. Device Manager now showed a new category and Arduino showed the **tools → ports** with **COM3**.

---

## The upload problem

Selected COM3 in Arduino, opened the Blink example from **File → Examples → 01.Basics → Blink**, clicked Upload.

```
error: 'LED_BUILTIN' was not declared in this scope
```

The ESP32 board package doesn't define LED_BUILTIN automatically. Replaced it with a direct pin number, GPIO2 is the standard onboard LED pin on DevKitC boards (according to the internet):

```cpp
#define LED_PIN 2

void setup() {
  pinMode(LED_PIN, OUTPUT);
}

void loop() {
  digitalWrite(LED_PIN, HIGH);
  delay(1000);
  digitalWrite(LED_PIN, LOW);
  delay(1000);
}
```

Clicked Upload again.

```
A fatal error occurred: Failed to connect to ESP32: Wrong boot mode detected (0x13)!
The chip needs to be in download mode.
```

The ESP32-DevKitC-32E needs to be manually put into download mode for uploading. The fix: click Upload, watch the output panel, and when `Connecting......` appears hold the **BOOT** button on the board until the progress bar starts moving. Release it once writing begins.

This worked. Upload completed successfully.

---

## The LED, not that kind of LED

Sketch uploaded. Board running. LED not blinking.

Tried inverting the logic in case the board used active-low:

```cpp
digitalWrite(LED_PIN, LOW);  // maybe this turns it ON?
```

Still nothing. I tried different pins, tried to really see a LED pin in the layout, but failed.

Added Serial output to confirm the code was actually running:

```cpp
void loop() {
  digitalWrite(LED_PIN, LOW);
  Serial.println("LED should be ON");
  delay(1000);
  digitalWrite(LED_PIN, HIGH);
  Serial.println("LED should be OFF");
  delay(1000);
}
```

Opened the Serial Monitor at 115200 baud. Messages printing perfectly. The code was running fine. The LED just wasn't responding.

Checked the official Espressif DevKitC-32E schematic again (this time not only the layout, but the descriptions), the genuine board has a power LED wired directly to 5V, not a user-controllable LED on any GPIO. Clone boards often add a GPIO2 LED but the real Espressif reference design doesn't have one. The board is from Botland and appears to be the genuine version.

---

## Summary, or What was done (less than half of what I planned, but oh well... next time)

Serial Monitor printing confirms:
- The board is recognised by Windows
- Arduino IDE can compile for ESP32
- The upload chain works (including the BOOT button workaround)
- The ESP32 is running code and communicating back over serial

That's everything needed to move on to sensor testing. The LED was never the point.

---

## Things that slowed my progress, and worth taking note on

- Most micro-USB cables in existence are charge-only. Test with a known data cable before debugging anything else.
- `LED_BUILTIN` is not defined for ESP32. Use the pin number directly.
- The DevKitC-32E needs the BOOT button held during upload. Click Upload, wait for `Connecting......`, hold BOOT, release when writing starts.
- The genuine Espressif DevKitC-32E has no user-controllable onboard LED. Serial Monitor is your indicator.

---

## Getting the ESP32 UN-Working (20-04-2026)

We had an election in the country that was really heated, my workplace was swamped with tasks, and I was not in the mood to work on this project for a few weeks.
I returned to it on the 20th of April, but I screwed it up.

I somehow pulled the microUSB connector off the ESP32...

Then in a panic mode, I tried to resolder it, and when I connected it to the computer again, it was not working at all...\
I checked again, and it seems like I messed up the soldering, and probably fried some of the microchips. I don't know, I do not have the tools to go so deep into the problem.\
I tried to create pictures to share, but phone cameras just can't focus well enough to the board, and from far away nothing is visible, from up-close it's just blurry.\
I tried and failed.

Next is ordering a new ESP32. \
...\
I've ordered, and after a few days I got back a reply, that they are out of stock, so I had to reorder from elsewhere.\
...\
Meanwhile something happened at Anthropic, and they are trying to charge me hundreds of euros for stuff I did not approve. I froze, and later trashed my card, ordered a new one, but Anthropic kept trying to charge 228.60 EUR, 114.30 EUR, and even charging my subscription fee way too early (after 6 days in the monthly plan). Zero support, chat bot generic replies. In the end I had to cancel my subscription.

It is a small setback, because I used claude code to clean my scripts, and create the manuals, and commenting for them (with human overview obviously, but it saved a lot of time), and I used it to help with finding the problems.\
If I get the new PCB, I will try without Claude code helping, and continue.

## Summary - One step forward, two steps back.

- Do not rush! Pulling the micro-USB out slowly and carefully saves you a lot of headaches. Alternatively buy an esp with usb-C port, it is less likely to get stuck.
- DO NOT RUSH! Using panic and adrenaline to quickly try to solder back something as small as a micro-USB connector is a really bad idea. Especially if you try to power it up without carefully checking the soldering job.
- While AI tools are making a lot of tedious tasks easier, and quicker, over reliance can lead to delays if something goes awry.

---

Next is sadly still testing the sensors one-by-one. (without Claude code)

---

*Concept: Auckland, June 2025. Build started: early 2026. Hungary. Database developer and administrator, 8 years. Named after Caelus, the Roman personification of the sky.*