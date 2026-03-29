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

## Story mode

### All of the parts arrived this week (26-03-2026)

https://github.com/SebastianSzat/Caleus_Node/blob/main/Hardware/First_mockup_parts_arrived.JPG
![First mockup parts arrived](/Hardware/First_mockup_parts_arrived.JPG?raw=true "First mockup parts arrived")

Next will be the software environment on the laptop when I have the time for it, testing the sensors one by one, and building the first "brain".

*Concept: Auckland, June 2025. Build started: early 2026. Hungary. Database developer and administrator, 8 years. Named after Caelus, the Roman personification of the sky.*