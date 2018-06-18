# iPython Notebook
The [notebook](./BinaryPlotter.md) loads the latest file from the PLC, and plots it using both pyplot, and plotly.

## Reading from the PLC using pyads
ADS is Beckhoff's protocol for reading and writing values from a PLC over the network. [pyads](https://pyads.readthedocs.io/en/latest/index.html) provides a convenient Python wrapper for this functionality.

The notebook first connects to the PLC, and reads the name of the most recent file to be written `MAIN.sFileName`. 
It then copies this over the network for processing. 
The PLC's network name and ADS address must be correctly configured.

The file is then unpacked using the `structure` module. This takes a format string describing the file's data structure, and converts it to a python list? or tuple?
