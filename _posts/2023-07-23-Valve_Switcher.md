---
layout: post
title: "Custom Valve Changer"

---

# The Project Setup

As part the [Raff Lab's](https://raff.lab.indiana.edu/index.html) field sampling campaiggn this summer, our group has been working with [Dr. Robert Hansen](https://www.qu.edu/faculty-and-staff/robert-hansen/) to measure fluxes of HONO, NO, and NO2 from the soil near the Indiana University Research and Teaching Preserve. 

Our experimental setup includes two NOx analyzers. Our group's instrument is coupled with a 5-way valve switcher programmed to sample 3 heights above the ground. Because our instrument is measuring HONO through catalytic nafion converters (see [this paper from our group](https://pubs.acs.org/doi/10.1021/acs.est.2c05944) for more) we needed additional valves in order to sample from six lines (two per position), adding complication to the procedure. Earlier this summer, I used the PACControl suite to program our instrument's valve switching assembly to correctly sample each height. 

Dr. Hansen's NOx analyzer is designed to measure NO using a similar methodology and was introduced as a background signal for each of our measurements. However, Dr. Hansen's assembly, although integrated into our tubing setup, lacked an automated valve changer to switch between levels automatically. Doing so would thus cost us days of lab time with manual switching. 

# The Solution

To solve this problem, I worked with Dr. Hansen, as well as Ph.D. candidates Liz Melissen and Evan Dalton, to design and build an electronic circuit, backed by a microcontroller, to control automated valve switching. 

Control is based around an Arduino Uno, which relies on serial communication to recieve control signals from the valve controlling computer. The Uno interfaces with a lightweight rust program which communicates via serial to keep the arduino clock synchronized with the controlling computer, whose clock tends to drift. The Uno is very straightforward to program, just using basic C++ to program the logic of the valve flow. 

The circuit design is also relatively straightforward. A pair of solid-state relays act as switches to complete the circuit for a 12v 1A power supply, driving each solenoid. Solid-state relays were used for this project because of problems with adequately sinking the power supply to ground with a transistor switch. SSRs were also an expedient solution which allowed us to more reliably operate in a continuous context. 

Because Dr. Hansen's setup does not measure through nafion, we were able to use two solenoid valves to control three heights. The valves were mounted on an existing plastic case with the wiring inside. This allowed compact transport to and from the field, and quick servicing should something go wrong.

# Outcomes

This was a great experience for me. As a first foray into circuit design, I learned a lot in the week that this project took to implement. This project has given me a great foundation on circuit interpretation and design, and more confidence in the future when I want to create a custom-designed instrument. 

