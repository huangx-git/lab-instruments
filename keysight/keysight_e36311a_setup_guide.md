# Keysight E36311A Power Supply Linux Control Guide

This document summarizes the steps required to configure, troubleshoot, and control a Keysight E36311A triple-output DC power supply on Ubuntu 20.04 using Python 3.8.

---

## 1. System & Hardware Specifications
* **Operating System:** Ubuntu 20.04
* **Python Version:** 3.8.10
* **Instrument:** Keysight E36311A Triple-Output DC Power Supply
* **USB Vendor ID / Product ID:** `0x2a8d` / `0x1002` (Decimal: `10893` / `4098`)
* **Discovered VISA Resource String:** `USB0::10893::4098::MY59001321::0::INSTR`

---

## 2. Resolving Linux USB Permissions (Udev Rules)
By default, Linux restricts non-root user access to USB instrument descriptors (Manufacturer, Product, Serial Number). To allow PyVISA-py to detect the device via USBTMC:

1. Create a udev rule file:
   ```bash
   sudo nano /etc/udev/rules.d/99-keysight.rules
   ```

2. Add the following rule line:
   ```text
   SUBSYSTEM=="usb", ATTRS{idVendor}=="2a8d", ATTRS{idProduct}=="1002", MODE="0666", GROUP="plugdev"
   ```

3. Save and close the file (`Ctrl+O`, `Enter`, `Ctrl+X`).

4. Reload rules and re-trigger the USB bus:
   ```bash
   sudo udevadm control --reload-rules
   sudo udevadm trigger
   ```
   *(Note: Unplug and re-plug the USB cable from the power supply to apply changes).*

---

## 3. Python Virtual Environment & Dependencies Setup
To ensure a clean, isolated package environment:

1. Install the Python 3.8 venv and low-level USB backend packages:
   ```bash
   sudo apt update
   sudo apt install python3.8-venv libusb-1.0-0
   ```

2. Create and activate your project virtual environment:
   ```bash
   cd ~/etroc2_pathfinder/lab-instruments
   python3.8 -m venv .venv
   source .venv/bin/activate
   ```

3. Install required Python packages:
   ```bash
   pip install --upgrade pip
   pip install pyvisa pyvisa-py pyusb
   ```

---

## 4. Resource Discovery & Verification
Run the following script to verify PyVISA-py detects the power supply:

```python
import pyvisa

rm = pyvisa.ResourceManager("@py")
print("Discovered Resources:", rm.list_resources())
```

**Expected Output:**
```text
Discovered Resources: ('USB0::10893::4098::MY59001321::0::INSTR', ...)
```

---

## 5. Control Script Template (`control.py`)
Below is a safe control script template for configuring channels, enabling outputs, and reading measurements:

```python
import time
import pyvisa

# Initialize PyVISA resource manager using the pure-Python backend
rm = pyvisa.ResourceManager("@py")
VISA_ADDRESS = "USB0::10893::4098::MY59001321::0::INSTR"

psu = None

try:
  print(f"Connecting to instrument at {VISA_ADDRESS}...")
  psu = rm.open_resource(VISA_ADDRESS)
  psu.timeout = 5000  # 5 seconds timeout

  # 1. Identify instrument
  print(f"Connected: {psu.query('*IDN?').strip()}")

  # 2. Reset and configure Channel 1 (3.3V, 0.5A limit)
  psu.write("*RST")
  time.sleep(0.5)

  psu.write("INST:NSEL 1")
  psu.write("VOLT 3.3")
  psu.write("CURR 0.5")

  # 3. Turn output ON
  psu.write("OUTP ON")
  print("Channel 1 output is ON.")

  # Read back actual measurements
  time.sleep(1.0)
  v_meas = float(psu.query("MEAS:VOLT?"))
  i_meas = float(psu.query("MEAS:CURR?"))
  print(f"Measured -> Voltage: {v_meas:.3f} V | Current: {i_meas:.3f} A")

  time.sleep(3.0)

except Exception as e:
  print(f"Error: {e}")

finally:
  if psu is not None:
    try:
      # SAFETY: Turn output OFF before closing connection
      psu.write("OUTP OFF")
      psu.close()
      print("Output disabled and connection closed safely.")
    except Exception:
      pass
```