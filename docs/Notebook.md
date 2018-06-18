# iPython Notebook
The [notebook](./BinaryPlotter.md) loads the latest file from the PLC, and plots it using both pyplot, and plotly.

## Reading from the PLC using pyads
ADS is Beckhoff's protocol for reading and writing values from a PLC over the network. [pyads](https://pyads.readthedocs.io/en/latest/index.html) provides a convenient Python wrapper for this functionality.

The notebook first connects to the PLC, and reads the name of the most recent file to be written `MAIN.sFileName`. 
It then copies this over the network for processing. 
The PLC's network name and ADS address must be correctly configured.

The file is then unpacked using the `struct` module. This takes a format string describing the file's data structure, and converts it to a tuple.

> A note on file format:  
> As the data on the PLC is stored by word, the size of `DCTIME64_UINT` structures is 2 words of 64bits each, or 16Bytes.
> However, the data within the structure only fills 64bits (`T_DCTIME64`) + 32bits (`UINT`) or 12Bytes.
> Therefore, the last 4Bytes of each datapoint are empty. These could be removed to reduce file size by 25%.

 ## Preparing the data
 Once the data is loaded, it can be formatted suitably for plotting.

 The timestamp in nanoseconds is converted to seconds, and the encoder value converted to degrees.

 The sample rate is calculated, and the minimum, maximum and mean values displayed. 
 If the PLC scans have remained determinstic (i.e. all scans have taken the same time) then the sample rate remains constant.

 The speed of the encoder is calculated by discrete derivative (i.e. forward difference). 
 A 100-point moving average is taken, to give a better graphical indication of speed.

 ## Plotting
 Plotting is provided using both `matplotlib` and `plotly`.

> Note:  
> Plotly doesn't like large datasets (500k points), which this code easily beats. 
> If plotting is too low, try uncommenting `plotter = go.Scattergl`.

![matplotlib](.\matplotlib.png)


![plotly](.\plotly.png)
