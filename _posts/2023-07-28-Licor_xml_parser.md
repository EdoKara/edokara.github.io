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

I used the `pyserial` package for serial communication. This part is as simple as can be; matching the port name and a timeout allows robust communication across the port:

```Python

portpath = "/dev/ttyUSB0"
ser = serial.Serial(portpath, timeout=1)

```


### Parsing the XML Packets

This part isn't quite so easy, but is helped greatly by the `beautifulsoup` package. I've used it before for web scraping, and this job was in general a lot more manageable than that one. 

After initializing the raw XML string into a BS4 interpreter object, I was able to query it like so:

```Python
co2 = conv_str_to_exp(str(i.find("co2").string)) 
```

The inner part of this comprehension (from an iterator) finds the "co2" tag in the XML object and unwraps the raw string. The outer function is custom and comes from a quirk of the li-cor's numeral notation, which is of the from "nnnne-n" or "nnnnen" (basically exponential notation). The conversion function works like this: 

```Python

def conv_str_to_exp(inputstr): #useful function, converts from 1.0380e-2 format to the actual float

    """
    This function takes a string or list-like of form "1.012908e10" and extracts just the exponent (the "10" from "e10" in this ex.) then 
    extracts just the pre-exponent. Then it exponentiates the pre-exponent to get a normal, python-readable number out.
    """

    exponentpattern = re.compile(r"(?<=(.e))(-?|\+?)\d+") #regex matches everything after an 'e' in a number
    preexppattern = re.compile(r"\d+\.?\d+(?=e|\se)") #matches everything up to an e in a number
    
    if type(inputstr) is str: #this if/elif logic lets you pass a list of strings or just one string to the function. 
        try:
            exp = exponentpattern.search(inputstr).group()
        except:
            exp = 1
        try:
            preexp = preexppattern.search(inputstr).group()
        except:
            preexp = 0
        
        finally: 
            exp = float(exp)
            preexp = float(preexp)

        return(preexp*(10**exp))

```

The regex is probably the most complicated part of this function, and took me a little while to fully validate. It matches either everything before an e in a numeral or everything after an e in a numeral, with the option for an intervening sign. Then it takes the group from the resulting regex, converts it to float, and exponentiates the value to get a normally-readable format. The try-except logic is to ensure that, if the query fails, a 0 will be output into the data (10^1). 

This logic is applied to each of the parameters of interest for the final CSVs, building to: 

```Python

for i in subset: #iterate over the subsets in the <data> tag
                #testing every conversion for a nonetype error;
                try:
                    
                    co2 = conv_str_to_exp(str(i.find("co2").string)) 
                except: #all try/excepts are the same; if the conversion fails return nan.
                    co2 = np.nan
                try:
                    
                    h2o = conv_str_to_exp(str(i.find("h2o").string)) 
                except:
                    h2o = np.nan
                try:
                    
                    celltemp = conv_str_to_exp(str(i.find("celltemp").string)) 
                except:
                    celltemp = np.nan
                try:
                    
                    cellpres = conv_str_to_exp(str(i.find("cellpres").string)) 
                except:
                    cellpres = np.nan

```

These values are then put together as elements in a row for the exporting dataframe like so:

```Python

line_to_read = pd.Series(data={"Datetime": datetime.now().strftime(format="%d/%m/%Y %H:%M:%S"), 
                      "co2":co2, 
                      "h2o":h2o, 
                      "celltemp":celltemp, 
                      "cellpres":cellpres}) 
                sl.append(line_to_read) #makes a pandas series out of the values from the searched tags in data. this is where I convert
                #number formats.

```

### Writing to CSV Regularly

The program buffers 20min of data in an internal `pandas` dataframe with the command

```python
for l in sl: #for each series (i.e. line of data reported):
                test = pd.concat([test, l.to_frame().T], ignore_index=True, axis=0, join='outer') # appends to the output file
                #print(test) #diagnostic print to see that things are working OK

```

I made this part a `for` loop in order to ensure that if the program ended up reading multiple lines at once, it would append all of them instead of dropping things. The export was accomplished with a custom function which was basically just a functionality wrapper to keep track of the program's flow a little better. 

