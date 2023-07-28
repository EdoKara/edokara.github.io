# LI-COR XML Parser 

One of my projects for the [Raff Lab](https://raff.lab.indiana.edu/index.html) was to update our team's strategy for data offload from our [LI-840A](https://www.licor.com/env/support/LI-840A/home.html) CO2 analyzer. 

The instrument itself works very well (calibration issues notwithstanding) but the lab's original means of accessing its data was to use LI-COR's proprietary software. The software was installed on a postively ancient (2004-era?) Dell laptop, pictured below next to my 2023 ASUS vivobook:

![Old Dell next to nearly-new Asus vivobook](/Assets/20230612_105331.jpg)

In addition to the sketchy screen, which didn't really want to stay readably bright, the case had a spot which electrocuted me when I touched it (left corner of the bottom half), leading me to some uncharitable conclusions about its reliability.

I decided to replace this computer and bring data reading to the lab's newer laptop, an ASUS laptop from circa 2015. In order to do this, I wrote a python script to read the LI-COR's serial output stream and parse it into CSV format.

## The Program

According to the OEM documentation from Li-Cor, the device outputs data via the serial port in XML format. I was able to get a sample of the format using [Tera Term](https://ttssh2.osdn.jp/index.html.en), and give the li-cor some slightly modified operating parameters. 

To read the data out, the program:

1. Establishes communication with the Li-Cor,
2. takes an XML packet and parses it to extract relevant information, and
3. writes that data to CSV at regular intervals. 

### Establishing communication


