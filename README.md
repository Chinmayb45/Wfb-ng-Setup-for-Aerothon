# 🚁 WFB-NG Autonomous Drone \& Video Streaming Architecture



> \*\*Deployment Context:\*\* Aerothon 26 Mission (Vicharak Axon \& Pixhawk)

> 

> \*\*System Overview:\*\* This architecture bridges a companion computer (Vicharak Axon) and a flight controller (Pixhawk) to a Ground Station using the WFB-NG long-range Wi-Fi protocol. It achieves simultaneous MAVLink routing for autonomous flight control and dual-view video processing (local VNC + remote WFB-NG encoding).



### 📡 Port Mapping Reference

|         Service          | IP Address  |   Port  | Protocol |              Purpose                 |

| \*\*MAVLink (WFB-NG)\*\* | `127.0.0.1` | `14560` |    UDP   | Downlink telemetry to Ground Station |

| \*\*MAVLink (Local)\*\*  | `127.0.0.1` | `14551` |    UDP   | Local autonomous mission script feed |

| \*\*MAVLink (QGC)\*\*    | `127.0.0.1` | `14550` |    UDP   | QGroundControl telemetry feed        |

| \*\*Video (Ingest)\*\*   | `127.0.0.1` | `5602`  |    UDP   | Drone camera encoding destination    |

| \*\*Video (Output)\*\*   | `127.0.0.1` | `5600`  |    UDP   | Ground Station GStreamer feed        |





## 🛠️ Phase 1: RTL8812EU Driver Compilation

To ensure monitor mode stability on the Vicharak Axon, the Wi-Fi driver must be patched and built from source using DKMS.



1. Clone the Source \& Patch

Pull the specific WFB-NG compatible driver source and prepare it for building:

```bash

git clone \[https://github.com/svpcom/rtl88x2eu.git](https://github.com/svpcom/rtl88x2eu.git)

cd rtl88x2eu

nano core/rtw\_mlme\_ext.c  # Apply monitor mode stability patch here

```



2. Force DKMS Build and Install

Register, build, and install the module into the active kernel tree:

```bash

sudo dkms add .

sudo dkms build rtl88x2eu/5.15.0.1

sudo dkms install rtl88x2eu/5.15.0.1

```



3. Load the Module

Wake up the driver and verify the interface (e.g., `wlxfc3...`) appears:

```bash

sudo modprobe 8812eu

iw dev

```






## 🔀 Phase 2: MAVLink Telemetry Routing

To prevent port collisions, `mavlink-router` duplicates the physical Pixhawk serial stream to feed both the local Python script and the WFB-NG transmitter.



1. Configure the Router

```bash

sudo nano /etc/mavlink-router/main.conf

```



2. Apply the Split Configuration

Replace the configuration with the following routing block:

```ini

\[General]

ReportStats=false



\[UartEndpoint pixhawk]

Device = /dev/ttyS4

Baud = 921600



\[UdpEndpoint wfb\_ng]

Mode = Normal

Address = 127.0.0.1

Port = 14560



\[UdpEndpoint python\_script]

Mode = Normal

Address = 127.0.0.1

Port = 14551

```



3. Apply Changes

```bash

sudo systemctl restart mavlink-router

```





## 🌉 Phase 3: WFB-NG Port Configurations

The over-the-air bridge must be mapped to intercept the specific telemetry and video ports.



1. Drone Configuration (`/etc/wifibroadcast.cfg`)

Configures the drone to listen for outgoing data:

```ini

\[common]

set\_nm\_unmanaged = False

wifi\_channel = 157

wifi\_region = 'BO'

bandwidth = 20



\[drone]

link\_domain = '1'



\[drone\_mavlink]

peer = 'listen://0.0.0.0:14560'



\[drone\_video]

peer = 'listen://0.0.0.0:5602'

```



2. Ground Station Configuration (`/etc/wifibroadcast.cfg`)

Configures the GS to push incoming radio data to the correct viewers:

```ini

\[common]

wifi\_channel = 157

wifi\_region = 'BO'

bandwidth = 20



\[gs]

link\_domain = '1'



\[gs\_mavlink]

peer = 'connect://127.0.0.1:14550'



\[gs\_video]

peer = 'connect://127.0.0.1:5600'

```



3. Restart WFB-NG Services

```bash

sudo systemctl restart wifibroadcast@drone  # Run on Drone

sudo systemctl restart wifibroadcast@gs     # Run on Ground Station

```





## 📷 Phase 4: Camera \& Video Encoding

To stream video efficiently, `camera.py` leverages hardware-accelerated GStreamer to pack RGB frames into H.264 UDP packets.



1. GStreamer Initialization

Add this to the Python OAK-D initialization block:

```python

wfb\_pipeline = (

&#x20;   "appsrc ! videoconvert ! video/x-raw,format=I420 ! "

&#x20;   "x264enc tune=zerolatency bitrate=2000 speed-preset=superfast ! "

&#x20;   "rtph264pay config-interval=1 pt=96 ! "

&#x20;   "udpsink host=127.0.0.1 port=5602 sync=false"

)

out\_wfb = cv2.VideoWriter(wfb\_pipeline, cv2.CAP\_GSTREAMER, 0, 30, (640, 360), True)

```



2. Pipeline Injection

Inside the main loop, write the fetched `rgbFrame` directly to the encoder:

```python

if out\_wfb and out\_wfb.isOpened():

&#x20;   out\_wfb.write(rgbFrame)

```




## 🚀 Phase 5: Mission Execution \& Dual Viewing



1. Launch the Autonomous Mission (Drone / Laptop 1)

Connect to the drone via VNC and execute the master mission script. This spawns the camera process and initiates local obstacle avoidance processing.

```bash

python3 vfh1.py

```



2. View Telemetry (Ground Station / Laptop 2)

Open \*\*QGroundControl\*\*. It will automatically detect the MAVLink stream pushed to `14550` and connect to the drone.



3. View Video Feed (Ground Station / Laptop 2)

Run the following command in the terminal to decode the low-latency H.264 video feed:

```bash

gst-launch-1.0 -v udpsrc port=5600 caps="application/x-rtp, media=(string)video, clock-rate=(int)90000, encoding-name=(string)H264, payload=(int)96" ! rtph264depay ! avdec\_h264 ! videoconvert ! autovideosink sync=false

```



