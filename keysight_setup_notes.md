# Keysight E36311A Linux Setup & Debug Notes

### 1. System & Hardware Details
* **OS:** Ubuntu 20.04
* **Python Version:** 3.8.10
* **Instrument:** Keysight E36311A Triple-Output DC Power Supply
* **USB Vendor/Product ID:** `2a8d:1002`
* **Discovered VISA Resource String:** `USB0::10893::4098::MY59001321::0::INSTR`

---

### 2. Resolving USB Permissions (Udev Rule)
* **Problem:** Linux kernel/udev blocked user-space access to USB device descriptors, causing PyVISA-py to only detect serial ports (`ASRL...`) and ignore the USBTMC interface.
* **Solution:** Created a dedicated udev rule file:
  1. File path: `/etc/udev/rules.d/99-keysight.rules`
  2. Rule content:
     ```text
     SUBSYSTEM=="usb", ATTRS{idVendor}=="2a8d", ATTRS{idProduct}=="1002", MODE="0666", GROUP="plugdev"
     ```
  3. Applied the changes and reloaded the USB bus:
     ```bash
     sudo udevadm control --reload-rules
     sudo udevadm trigger
     ```
     *(Followed by unplugging and re-plugging the USB cable).*

---

### 3. Python Virtual Environment & Dependencies
* **Problem:** Creating a virtual environment (`python3 -m venv .venv`) failed due to a missing `ensurepip` package (`python3.8-venv`).
* **Solution:** 
  1. Installed the system venv package for Python 3.8:
     ```bash
     sudo apt install python3.8-venv
     ```
  2. Created and activated a clean virtual environment inside the project folder:
     ```bash
     python3.8 -m venv .venv
     source .venv/bin/activate
     ```
  3. Installed PyVISA and backend dependencies:
     ```bash
     pip install --upgrade pip
     pip install pyvisa pyvisa-py pyusb
     ```

---

### 4. Verification & Testing Script
* **Resource Discovery Script:**
  ```python
  import pyvisa

  rm = pyvisa.ResourceManager("@py")
  print("Discovered Resources:", rm.list_resources())

