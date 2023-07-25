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

## Programming

The control program is an interface between an Arduino Uno and a Rust binary running on Windows. The tasks required to get the program working were:

1. Establish communication between my Rust environment and the Arduino
2. Write the logic synchronizing the valve switcher with system time, and therefore the rest of the instrument ecosystem.
3. Produce a set of flags to record when an elevation was switched to, and which elevation it was at during each window of time. 
4. Port the linux compile to a Windows target so that it will run on the NOxWerx computer. 

### Establishing Communication

Communication used the `serialport` [rust crate](https://github.com/serialport/serialport-rs), which is similar in API to PySerial. Establishing a connection is as easy as entering the parameters like so: 

```Rust
let builder = serialport::new("/dev/ttyACM0", 9600).timeout(Duration::from_millis(3500)).data_bits(DataBits::Eight).stop_bits(StopBits::One);

let mut port = builder.open().expect("failed to connect."); 
```

The major issue establishing a connection is on the Arduino's end. The uno I used takes ~3s to reset and begin communicating after a new serial link is established, which means that the rust code has to wait to send instructions. 

### Valve Logic

Valve logic was relatively straightforward and mirrored that of the main valve switcher in Opto. I used the `time` package in `std` to get system time from the PC, allowing me to operate off the same 5-minute start logic as the other valve changers on the system. I used a set of switching flags, as opposed to relying only on timing logic, to indicate when a loop of the program is complete: 

```Rust
while true {
    //below func gets system time.
    let time_startloop:u64 = SystemTime::now()
    .duration_since(SystemTime::UNIX_EPOCH).unwrap()
    .as_secs();

    let mut timing_flags: [i32; 3] = [0,0,0];
    
    //second while loop for the interior stuff
    while timing_flags != [1,1,1] {
        let time_now:u64 = SystemTime::now()
        .duration_since(SystemTime::UNIX_EPOCH).unwrap()
        .as_secs();

        if time_now - time_startloop == 0 {
            sendmsg(1);
            timing_flags[0] = 1;
            thread::sleep(Duration::from_millis(5*60*1000));
        }    

        if time_now - time_startloop == 5*60 {
            sendmsg(2);
            timing_flags[1] = 1;
            thread::sleep(Duration::from_millis(5*60*1000));
        }
        
        if time_now - time_startloop == 10*60 {
            sendmsg(0);
            timing_flags[2] = 1;
            thread::sleep(Duration::from_millis(5*60*1000));
        }

    }
}
```

The differencing approach was a relatively fast way to get a relative system time without offsetting from `UNIX_EPOCH`. 

### Flagging



### Cross-Compiling

For cross-compilation, I used the rust `cross` [package](https://github.com/cross-rs/cross/tree/main), which uses `docker` to compile into other distros and OSes from the command line, with cloned interface from `cargo`. 



## Circuit Design
The circuit design is also relatively straightforward. A pair of solid-state relays act as switches to complete the circuit for a 12v 1A power supply, driving each solenoid. Solid-state relays were used for this project because of problems with adequately sinking the power supply to ground with a transistor switch. SSRs were also an expedient solution which allowed us to more reliably operate in a continuous context. 

Because Dr. Hansen's setup does not measure through nafion, we were able to use two solenoid valves to control three heights. The valves were mounted on an existing plastic case with the wiring inside. This allowed compact transport to and from the field, and quick servicing should something go wrong.

# Outcomes

This was a great experience for me. As a first foray into circuit design, I learned a lot in the week that this project took to implement. This project has given me a great foundation on circuit interpretation and design, and more confidence in the future when I want to create custom-designed tools and circuits. 

