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

{% highlight Rust %}
let builder = serialport::new("/dev/ttyACM0", 9600).timeout(Duration::from_millis(3500)).data_bits(DataBits::Eight).stop_bits(StopBits::One);

let mut port = builder.open().expect("failed to connect."); 
{% endhighlight %}

The major issue establishing a connection is on the Arduino's end. The uno I used takes ~3s to reset and begin communicating after a new serial link is established, which means that the rust code has to wait to send instructions. 

### Valve Logic

Valve logic was relatively straightforward and mirrored that of the main valve switcher in Opto. I used the `time` package in `std` to get system time from the PC, allowing me to operate off the same 5-minute start logic as the other valve changers on the system. I used a set of switching flags, as opposed to relying only on timing logic, to indicate when a loop of the program is complete: 

```Rust
use std::{thread, 
    time::{Duration, SystemTime}, 
    io::Result};
use serialport::{DataBits, StopBits};
use chrono::{NaiveDateTime, Datelike, Timelike};


pub const TIME_OFFSET: u64 = 60*11 + 44; //offset between the NOxWerx and the computer this is running on
pub const DISPLAYTIME_OFFSET: u64 = 60*60*4; //offset btw EST and UTC (?) 

macro_rules! get_time { //macro gets the current system time and converts it into a seconds figure
    () => {SystemTime::now()
        .duration_since(SystemTime::UNIX_EPOCH)
        .unwrap()
        .as_secs() + TIME_OFFSET
    };
}

macro_rules! get_time_display { //takes a time object (even get_time!() from above) and converts it to a human-readable datetime obj
    ($raw_time:ident) => { 
        NaiveDateTime::from_timestamp_opt(
        ($raw_time - DISPLAYTIME_OFFSET) as i64,0_u32)
    .unwrap()
        
    };
}

macro_rules! timestamp { //must be used with a NativeDateTime object to behave as expected
    ($level:literal, $time_obj:ident) => {
        println!("{}, {:02}/{:02}/{:02} {:02}:{:02}:{:02}", $level,
                    $time_obj.year(), $time_obj.month(), $time_obj.day(),
                    $time_obj.hour(), $time_obj.minute(), $time_obj.second())
    };
}

pub fn main() {
    println!("Level, Datetime");
        
    setup(60*5); //starts it on the next 5mins. Accounts for the time it takes for\
            // the port to reset as well.
    


    loop { //this is the top-level loop for the program

    
        

//this is the loop the valve switching operates in 
    //below func gets system time.
    let time_startloop: u64 = get_time!();
    let mut timing_flags: [i32; 3] = [0,0,0]; //flagging var to check when all 3 levels are cycled thru
    let routime:u64 = 60*5; //var to define the run time in 1 place
    let max_buffer:u64 = 25; //maximum error acccepted before a condition is considered skipped
    
    //second while loop for the interior stuff
    while timing_flags != [1,1,1] { //iterate any time the flags aren't all set

        let time_now:u64 = get_time!(); //get time to compare to the loop start time
        
        let displaytime: NaiveDateTime = get_time_display!(time_now); //make a display-friendly version 

        if  time_now-time_startloop <= max_buffer //criteria for the first level
            && timing_flags[0]!=1 {

                match sendmsg(1){ //either send the message successfully or return an error 
                    Ok(_usize) =>(),
                    Err(_e) => println!("Failed to write!")
                };
                timing_flags[0] = 1; //update the timing flag 
                timestamp!(1, displaytime); //print out the time that it switched at 
                thread::sleep(Duration::from_millis(routime*1000)) //sleep for 5min worth of milliseconds
            }    

        if time_now - time_startloop >= routime //criteria for the 2nd valve switch
        && time_now-time_startloop <= (routime)+max_buffer
        && timing_flags[1]!=1 {

            match sendmsg(2){ //same sequence as before
                Ok(_usize) =>(),
                Err(_e) => println!("Failed to write!")
            };
            timing_flags[1] = 1;
            timestamp!(2, displaytime);
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
            timestamp!(3, displaytime);
            thread::sleep(Duration::from_millis(routime*1000))
        }

        if time_now - time_startloop >= routime*3 + 15 &&
        timing_flags !=[1,1,1]{
            timestamp!("overtime", displaytime);
            sendmsg(0).expect("failed!");
            break
        }
        
    }
}
    }

    
pub fn sendmsg(input:u8) -> Result<usize> {    
   //!
   //! # Send Message
   //! 
   //! this function sends the Arduino a message. Because the wait times between messages are so long,
   //! the function is made so that it opens a new serial connection every time it needs to talk to the arduino.
   //! If sending more regular signals, you need to modify this so that the port sits open (probably at the top of the loop)
   //! because the arduino has about a ~2.5 second lag between when you start opening the serial port and when it can
   //! take messages. There is also some timeout on the computer's side of the connection, so if the input is 
   //! slow enough you'll run up against that constraint and have to open the port every time. 


    let builder = serialport::new("COM5", 9600) //params to tune for the Arduino port
        .timeout(Duration::from_millis(3000)) //3s timeout 
        .data_bits(DataBits::Eight)
        .stop_bits(StopBits::One);

    let mut port = builder.open().expect("failed to connect."); //

    
    let binding: String = input.to_string();
    let writebyte:&[u8] = binding.as_bytes();

    port
    .write(writebyte)

}
        
pub fn setup(routime:u64) { 

    //! # Setup
    //! 
    //! this func waits the specified amount of time before starting the rest of the program. Very small wrapper. 

    let mut startflag:u8 = 0;

    while startflag == 0 {
    let time_now:u64 = SystemTime::now()
        .duration_since(SystemTime::UNIX_EPOCH)
        .unwrap()
        .as_secs()+TIME_OFFSET;

    if time_now % (routime) == 0 {
        startflag = 1;}
        else {startflag = 0;}
    }
}

```

The differencing approach was a relatively fast way to get a relative system time without offsetting from `UNIX_EPOCH`. 

There are some clear improvements which could be made to this code. I think that it would be better to run this as a state machine instead of a set of conditionals, as this would limit unexpected behavior and allow for better error handling. There is also very likely to be a way to systematize the construction of a valve switching logic with a standard set of features such that this could be extended into a large domain of simple cases easily. I would likely use a rust macro for this, but it would be a pretty advanced job since it would rely on programmatic code expansion. 

### Flagging

Flagging required some substantial modifications to work. I initially wrote a function using the `csv` package to send the flags out to a filepath. `write_flags` is only handling the logic of passing the values, and timing is handled eleswhere. I used the `Result <()>` type to allow some amount of error handling in the function. This is, however, probably one of the more stable parts of the program, whereas timing is a riskier and more complicated task. 
```Rust

macro_rules! timestamp { //must be used with a NativeDateTime object to behave as expected
    ($level:literal, $time_obj:ident) => {
        println!("{}, {:02}/{:02}/{:02} {:02}:{:02}:{:02}", $level,
                    $time_obj.year(), $time_obj.month(), $time_obj.day(),
                    $time_obj.hour(), $time_obj.minute(), $time_obj.second())
    };
}

```

### Cross-Compiling

For cross-compilation, I used the rust `cross` [package](https://github.com/cross-rs/cross/tree/main), which uses `docker` to compile into other distros and OSes from the command line, with cloned interface from `cargo`. 

## Circuit Design
The circuit design is also relatively straightforward. A pair of solid-state relays act as switches to complete the circuit for a 12v 1A power supply, driving each solenoid. Solid-state relays were used for this project because of problems with adequately sinking the power supply to ground with a transistor switch. SSRs were also an expedient solution which allowed us to more reliably operate in a continuous context. A circuit diagram is included below: 

![Valve changer circuit diagram](/Assets/Valve_Changer_circuit_diagram.png)

Because Dr. Hansen's setup does not measure through nafion, we were able to use two solenoid valves to control three heights. The valves were mounted on an existing plastic case with the wiring inside. This allowed compact transport to and from the field, and quick servicing should something go wrong.

# Outcomes

This was a great experience for me. As a first foray into circuit design, I learned a lot in the week that this project took to implement. This project has given me a great foundation on circuit interpretation and design, and more confidence in the future when I want to create custom-designed tools and circuits. 

Furthermore, this project has given me a lot of confidence in Rust programming. I ended up using some of the more foreign (at least to a Python user) features of Rust in this project and gained a great deal of understanding from the process. 

