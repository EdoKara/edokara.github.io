---
layout: post
title: "Custom Valve Changer"

---

# The Project Setup

As part the [Raff Lab's](https://raff.lab.indiana.edu/index.html) field sampling campaiggn this summer, our group has been working with [Dr. Robert Hansen](https://www.qu.edu/faculty-and-staff/robert-hansen/) to measure fluxes of HONO, NO, and NO2 from the soil near the Indiana University Research and Teaching Preserve. 

Our experimental setup includes two NOx analyzers. Our group's instrument is coupled with a 5-way valve switcher programmed to sample 3 heights above the ground. Because our instrument is measuring HONO through catalytic nafion converters (see [this paper from our group](https://pubs.acs.org/doi/10.1021/acs.est.2c05944) for more) we needed additional valves in order to sample from six lines (two per position), adding complication to the procedure. Earlier this summer, I used the PACControl suite to program our instrument's valve switching assembly to correctly sample each height. 

Dr. Hansen's NOx analyzer is designed to measure NO using a similar methodology and was introduced as an independent background signal for each of our measurements. However, Dr. Hansen's assembly, although integrated into our tubing setup, lacked a valve changer to switch between levels automatically. Doing so would thus cost us days of lab time with manual switching. 

# The Solution

To solve this problem, I worked with Dr. Hansen, as well as Ph.D. candidates Liz Melissen and Evan Dalton, to design and build an electronic circuit, backed by a microcontroller, to automate the switching process.

The control program is hosted on the same computer that Dr. Hansen's instruments are reading to. That computer's time must be offest to account for the lag between the real time (the host computer which is connected to the internet) and the time on the other NOxWerx, which controls our stack of valve switchers to synchronize to, and which the OPTO IC syncs to. 

## Programming

The control program is an interface between an Arduino Uno and a Rust binary running on Windows. The tasks required to get the program working were:

1. Establish communication between my Rust program and the Arduino
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
loop {

    let fstarttime:u64 = get_time!();

    let displaystring = format!("{:02}{:02}{:02}{:02}{:02}", //format YYMMDDHHMM
        get_time_display!(fstarttime).year(), 
        get_time_display!(fstarttime).month(),get_time_display!(fstarttime).day(),
        get_time_display!(fstarttime).hour(),get_time_display!(fstarttime).minute()
        );
        
    let filenamestr: String = format!("rob_noxbox_{}.csv", &displaystring); //concatenating the filenamestr
    let file = filepath.join(&filenamestr); //final filename

    while get_time!() - fstarttime < file_write_interval {
    //below func gets system time.
    let time_startloop: u64 = get_time!();
    let mut timing_flags: [i32; 3] = [0,0,0]; //flagging var to check when all 3 levels are cycled thru
    let routime:u64 = 60*5; //var to define the run time in 1 place
    let max_buffer:u64 = 25; //maximum error acccepted before a condition is considered skipped
    
    //second while loop for the interior stuff
    while timing_flags != [1,1,1] { //iterate any time the flags aren't all set

        let time_now:u64 = get_time!();
        
        let displaytime: NaiveDateTime = get_time_display!(time_now);

        if  time_now-time_startloop <= max_buffer
            && timing_flags[0]!=1 {

                match sendmsg(1){
                    Ok(_usize) =>(),
                    Err(_e) => println!("Failed to write!")
                };
                timing_flags[0] = 1;
                println!("Level 1 at {:02}/{:02}/{:02} {:02}:{:02}:{:02}",
                displaytime.year(), displaytime.month(), displaytime.day(),
                displaytime.hour(), displaytime.minute(), displaytime.second());
                _ = write_flags(routime, &file, 1);
                thread::sleep(Duration::from_millis(routime*1000))
            }    

        if time_now - time_startloop >= routime 
        && time_now-time_startloop <= (routime)+max_buffer
        && timing_flags[1]!=1 {

            match sendmsg(2){
                Ok(_usize) =>(),
                Err(_e) => println!("Failed to write!")
            };
            timing_flags[1] = 1;
            println!("Level 2 at {:02}/{:02}/{:02} {:02}:{:02}:{:02}",
                displaytime.year(), displaytime.month(), displaytime.day(),
            displaytime.hour(), displaytime.minute(), displaytime.second());

            _ = write_flags(routime, &file, 2);
            thread::sleep(Duration::from_millis(routime*1000))
        }
        
        if time_now - time_startloop >= routime*2
        && time_now - time_startloop <=(routime*2 )+max_buffer
        && timing_flags[2]!=1 {

            match sendmsg(0){
                Ok(_usize) =>(),
                Err(_e) => println!("Failed to write!")
            };
            timing_flags[2] = 1;
            println!("Level 3 at {:02}/{:02}/{:02} {:02}:{:02}:{:02}",
                displaytime.year(), displaytime.month(), displaytime.day(),
            displaytime.hour(), displaytime.minute(), displaytime.second());
            _ = write_flags(routime, &file, 3);
            thread::sleep(Duration::from_millis(routime*1000))
        }

        if time_now - time_startloop >= routime*3 + 15 &&
        timing_flags !=[1,1,1]{
            println!("Overtime at {:02}/{:02}/{:02} {:02}:{:02}:{:02}",
                displaytime.year(), displaytime.month(), displaytime.day(),
            displaytime.hour(), displaytime.minute(), displaytime.second());
            sendmsg(0).expect("failed!");
            break
        }
        
    }
}
    }
```

The differencing approach was a relatively fast way to get a relative system time without offsetting from `UNIX_EPOCH`. 

There are some clear improvements which could be made to this code. I think that it would be better to run this as a state machine instead of a set of conditionals, as this would limit unexpected behavior and allow for better error handling. There is also very likely to be a way to systematize the construction of a valve switching logic with a standard set of features such that this could be extended into a large domain of simple cases easily. I would likely use a rust macro for this, but it would be a pretty advanced job since it would rely on programmatic code expansion. 

### Flagging

Flagging required some substantial modifications to work. I wrote a function using the `csv` package to send the flags out to a filepath. `write_flags` is only handling the logic of passing the values, and timing is handled eleswhere. I used the `Result <()>` type to allow some amount of error handling in the function. This is, however, probably one of the more stable parts of the program, whereas timing is a riskier and more complicated task. 
```Rust

fn write_flags(routime:u64, 
    file:&Path, position:u8)-> Result<()> {

    let mut wtr = csv::Writer::from_path(file)?; //pass it to the writer function

        let now = get_time!();
        let displaynow = get_time_display!(now);

        wtr.write_record(vec![displaynow.to_string(), position.to_stcontrol automated valve switching. ring()])?;

    wtr.flush()?;
    Ok(())
}

```

### Cross-Compiling

For cross-compilation, I used the rust `cross` [package](https://github.com/cross-rs/cross/tree/main), which uses `docker` to compile into other distros and OSes from the command line, with cloned interface from `cargo`. 

## Circuit Design
The circuit design is also relatively straightforward. A pair of solid-state relays act as switches to complete the circuit for a 12v 1A power supply, driving each solenoid. Solid-state relays were used for this project because of problems with adequately sinking the power supply to ground with a transistor switch. SSRs were also an expedient solution which allowed us to more reliably operate in a continuous context. 

Because Dr. Hansen's setup does not measure through nafion, we were able to use two solenoid valves to control three heights. The valves were mounted on an existing plastic case with the wiring inside. This allowed compact transport to and from the field, and quick servicing should something go wrong.

# Outcomes

This was a great experience for me. As a first foray into circuit design, I learned a lot in the week that this project took to implement. This project has given me a great foundation on circuit interpretation and design, and more confidence in the future when I want to create custom-designed tools and circuits. 

Furthermore, this project has given me a lot of confidence in Rust programming. I ended up using some of the more foreign (at least to a Python user) features of Rust in this project and gained a great deal of understanding from the process. 

