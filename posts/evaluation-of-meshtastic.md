---
create_date: 2024-05-17 00:01:46+00:00
edit_date: 2024-05-17 00:01:46+00:00
build_date: 2024-05-19 21:42:15+00:00
title: Evaluation of Meshtastic
link: https://leonid.belyaev.systems/posts/evaluation-of-meshtastic.html
---


I've been interested in computer communication by means of radio using lower power over longer distances for a while. This is a primary reason for my getting a Technician-class Ham Radio license; [Rag-Chewing](https://www.onallbands.com/word-of-the-day-rag-chewing/) was of less interest to me.

[HF](https://en.wikipedia.org/wiki/High_frequency), however, has limitations. While [QRP distance records](http://naqcc.info/qrpworks.html) can be staggeringly impressive,
HF antennae tend to be rather large. When operating in the field, a [random wire](https://en.wikipedia.org/wiki/Random_wire_antenna) approach combined with antenna tuning is viable, although somewhat cumbersome.

Even if HF fulfilled all technical requiements, it is also illegal to encrypt transmissions on it as a ham operator. This makes it unsuitable for many practical purposes.

Enter [Meshtastic](https://meshtastic.org/), running on LoRA, in turn running on 915Mhz in the USA.
Like 2.4Ghz WiFi, this is one of the (more-or-less) unregulated [ISM](https://en.wikipedia.org/wiki/ISM_radio_band) bands, originally reserved legally for equipment like microwave ovens, and therefore, transmission in these bands can be done without a government-issued license, and encryption is legal.
Range is extended in the meshtastic network by meshing of nodes, or as in Ham Radio, internet-based repeaters, running on [MQTT](https://en.wikipedia.org/wiki/MQTT) in this case.
The current node-to-node transmission distance record is [254km==158mi](https://meshtastic.org/docs/overview/range-tests/).

LoRA is a proprietary radio communication technique that enables transmission over longer distances using lower power levels.
Explaining it is beyond the scope of this post. For an explanation of how it works mathematically, see [this video](https://www.youtube.com/watch?v=jHWepP1ZWTk).

With graduating school and enterfering the workforce, I've had the time and money to finally try it out for myself.
It follows along a general trend of my technological development -- implementing things in a way that I can control, on my own time, learning a lot along the way. While cellular networking has made access to the internet almost anywhere while out and about trivial, this service presents an opportunity to operate my own "cellular network" without breaking the law. More fantastically, it presents to me a means of ultimately escaping the grip of subscription-based cellular plans and the somewhat "[un-computer](https://www.youtube.com/watch?v=zfR_Jj4grZE)" Android and iOS operating systems.

#### Explaining Channels and Channel Setup

Like IRC or Slack, a channel in Meshtastic is not physical, but virtual. Creating an encrypted logical channel takes a bit of setup.

Meshtastic radios generally have eight channel slots, which are zero-indexed, with the first channel of index zero coming as default in the firmware. This channel uniquely has the PRIMARY role, which implements the telemetry and position-reporting [APRS](https://en.wikipedia.org/wiki/Automatic_Packet_Reporting_System)-like features of Meshtastic. 

This channel should be considered public. It is secured using only a short publically known pre-shared key.

Radios in the mesh must be set to the same LoRA frequency slot (the physical channel, in this case) in order to relay one another's messages. By default, the primary channel's name is hashed to set this frequency slot. For this reason, keeping the primary channel as the default public one is a reasonable choice for getting-going.

For private communication, then, a simple configuration is to create one (or more) secondary channels in the index slots 1, 2, etc, and keep the PRIMARY pre-baked channel as-is. To do this with the Meshtastic CLI, with factory-new radios connected:

```bash
# With the first radio connected
meshtastic --ch-add <NAME>
meshtastic --ch-set name <SAME_NAME> --ch-set psk random --ch-index 1

# Take note of the generated PSK near the bottom
meshtastic --info

# With the nth radio connected
meshtastic --ch-add <SAME_NAME>
# Enter the PSK found to be generated on the first radio with meshtastic --info
meshtastic --ch-set name <SAME_NAME> --ch-set psk base64:<GENERATED_PSK> --ch-index 1
```

More information about channel configuration [here](https://meshtastic.org/docs/configuration/radio/channels/). It is possible to create a more secure configuration without position sharing if desired.


#### Range Test Setup

My test setup was as follows: my rack-mounted desktop computer was connected to a radio base station (a [WisBlock Meshtastic Starter Kit](https://store.rakwireless.com/products/wisblock-meshtastic-starter-kit?variant=43683420438726)) by USB serial cable. While on foot, I used a [T-Beam SUPREME](https://www.lilygo.cc/products/t-beamsupreme-m?variant=43067943977141) to attempt to contact my home node. Both systems ran [Linx 916Mhz Antennae](https://www.digikey.com/en/products/detail/te-connectivity-linx/ant-916-cw-hw/809424).

My objective was to not only reach my home node from the mobile radio, but also for the home node to reach the mobile radio with a contentful response. A simple bot was programmed using [this Rust meshtastic library](https://docs.rs/meshtastic/latest/meshtastic/) to respond to any sender of the string `/weather` on private channel with a formatted `curl` of `https://wttr.in`, forming a sort of poor-man's NOAA service (which pool lifeguards might find as the sole entertainment on their walkie-talkies during an eleven-hour shift). Operating the mobile radio was done using a [GPD Pocket 3](https://www.gpd.hk/gpdpocket3) portable PC, USB cable, and Meshtastic CLI.

Because I knew that `$CITY` hosts a meshtastic node in its tallest building, I knew that positioning my home node such that it could hit this building would dramatically improve connectivity to and from my mobile radio. In my case, it was necessary to move the home node just a meter or so from out of my internal brick wall and barred windows into a patio outside. I didn't have true line-of-sight to the elevated node, but this was sufficient to get reliable pings to it and others in the neighborhood -- otherwise there was nothing.

#### Range Test Results

With line of sight to the elevated node, I was able to hit my home node and receive a reply within 1~ second as far away as one mile distance between. I didn't put in the time to walk further out, but I would imagine that yet greater range would be totally possible, so long as a good line-of-sight was maintained -- I found that even limited obstructions could reduce reliability, sometimes down to zero. Range tests from operation at home definitely suggest this, as I can hit other remote nodes at a greater distance without doing anything special. Because of the importance of line of sight (and a lack of [noisy SPMS power supplies](https://en.wikipedia.org/wiki/Switched-mode_power_supply), etc), open spaces like parks and rivers open up connectivity more so relative to more settled areas.

#### Conclusion

While programmed Meshtastic communication won't outright replace my smartphone and cell service anytime soon, I'd like to continue working in this ecosystem.

Among other things, I'd like to build an enclosed weatherproof home node, potentially using solar panels to stay self-sufficient. This way I could have a permanent home node I wouldn't be restricted to operating only on dry days at the cost of an open window.

Something that I would like to explore is implementing a bridge for my IM
messenger of choice, or perhaps email. Meshtastic cannot support many users in constant chatty IM, but it could be neighborly and usable for sending a simple "BRB" type message.

Some Meshtastic systems support [codec2](https://www.rowetel.com/wordpress/?page_id=452)-encoded audio messages.

Brutal lossy image compression by means of LLMs and GenAI is another potential, crushing images down to a semantic description and re-generating them on the other end.

With a Meshtastic node hooked up to the internet and my home network, a variety of other automations could be developed. The key is working within the limits of the [low datarate](https://meshtastic.org/docs/overview/radio-settings/#data-rates).

#### Further Reading 

[Meshtastic Mesh Algorithm](https://meshtastic.org/docs/overview/mesh-algo/)