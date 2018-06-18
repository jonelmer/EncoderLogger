

```python
# Get the latest fie from the PLC

import pyads

plc_name = "//CX-31162A"
plc = pyads.Connection('5.49.22.42.1.1', 851)

plc.open()
remote_file = plc.read_by_name('MAIN.sFileName', pyads.PLCTYPE_STRING)
plc.close()

import os.path
import shutil

d, f = os.path.splitdrive(remote_file)
h, t = os.path.split(remote_file)

filename = shutil.copyfile(os.path.join(plc_name, f[1:]), t)

if filename:
    print("Copied {} from the PLC".format(filename))
```

    Copied 2018-06-18 09-48-37.bin from the PLC
    


```python
import struct

#filename = "2018-06-14 14-18-13.bin"

# Set up the structure to read:
struct_fmt = 'QI4x' # T_DCTIME64 = Q, UINT = I, 4 padding bytes
struct_len = struct.calcsize(struct_fmt)
struct_unpack = struct.Struct(struct_fmt).unpack_from

# Load the file
datapoints = []
with open(filename, "rb") as f:
    #for i in range(10):
    while True:
        data = f.read(struct_len)
        if not data: break
        s = struct_unpack(data)
        datapoints.append(s)
        
print("Got {} samples".format(len(datapoints)))
```

    Got 264438 samples
    


```python
import numpy as np

# Trim the data?
#datapoints = datapoints[33500:36500]
#datapoints = datapoints[:100000]

(time_ns, enc_raw) = zip(*datapoints)

# Normalise data to the first sample - everything is now relative to [0]
enc_deg = [(r - enc_raw[0])/4 for r in enc_raw]
time_ns = [(t-time_ns[0]) for t in time_ns]
time_s = [t/1e9 for t in time_ns]

# Sample rate
rate_ns = [(u - t) for (t, u) in zip(time_ns, time_ns[1:])]
print("Sample rate\n  Min:  {}ns\n  Mean: {}ns\n  Max:  {}ns\n".format(np.min(rate_ns), np.mean(rate_ns), np.max(rate_ns)))

# Moving avrage function
def moving_average(a, n=3) :
    ret = np.cumsum(a, dtype=float)
    ret[n:] = ret[n:] - ret[:-n]
    return ret[n - 1:] / n

# Speed
speed = [f - e for (e, f) in zip(enc_deg, enc_deg[1:])]
speed = moving_average(speed, 100)

# Info string
info = "No. samples: {}\nSample rate: {}s".format(len(datapoints),np.mean(rate_ns)/1e9)
print(info)
```

    Sample rate
      Min:  100000ns
      Mean: 100000.0ns
      Max:  100000ns
    
    No. samples: 264438
    Sample rate: 0.0001s
    


```python
import matplotlib.pyplot as plt

# Plotting
fig, (ax1, ax2) = plt.subplots(2, 1, True)

ax1.plot(time_s, enc_deg, label="Angle")
ax2.plot(time_s[:len(speed)], speed, 'r-', label="Speed")

ax1.set_ylabel("Angle (°)")
ax2.set_xlabel("Time (s)")
ax2.set_ylabel("Speed (°/s)")

ax2.text(0.05, 0.95, info, transform=ax2.transAxes, fontsize=10, verticalalignment='top')

# Legend
#lines, labels = ax1.get_legend_handles_labels()
#lines2, labels2 = ax2.get_legend_handles_labels()
#ax2.legend(lines + lines2, labels + labels2, loc=4)

plt.show()
```


![matplotlib](.\matplotlib.png)



```python
from plotly import tools
import plotly.plotly as py
import plotly.graph_objs as go

plotter = go.Scatter
#plotter = go.Scattergl

data = [ 
    plotter(x=time_s, y=enc_deg, name="Angle"), 
    plotter(x=time_s, y=speed,   name="Speed", yaxis="y2", line={'color':('rgb(205, 12, 24)')})
    ]

layout = go.Layout(
    width = 800,
    height = 600,
    title = "Encoder Data",
    xaxis = dict(
        title = "Time (s)",
        rangeslider=dict()
    ),
    yaxis = dict(
        title = "Angle (°)",
    ),
    yaxis2 = dict(
        title = "Speed (°/s)",
        overlaying='y',
        side='right'
    ),
    showlegend= False,
    annotations=[
        dict(
            x=0.05,
            y=0.05,
            align='left',
            showarrow=False,
            text=info,
            xref='paper',
            yref='paper'
        ),]
)

fig = go.Figure(data=data, layout=layout)
py.iplot(fig, filename="encoder_data")
```



[View the graph here  
![Plotly graph](https://plot.ly/~jelmerstfc/6.jpg)](https://plot.ly/~jelmerstfc/6)

