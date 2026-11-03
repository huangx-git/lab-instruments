# Keysight E36311A over USB: step-by-step bring-up (Python 3.8)

Do the steps in order and stop at the first one that fails. Each step is small and
tells you what a good result looks like. Steps 1-4 only read; nothing changes on the supply
until step 6.

Run the Python steps in an interactive session (`python3`), so every line can be checked
before the next one.

---

## Step 0. Install

```bash
python3 --version          # expect 3.8.10
pip install pyvisa pyvisa-py pyusb
```

If `pip` complains about the Python version: `pip install "pyvisa<1.15" pyvisa-py pyusb`.

If `pyusb` later cannot find a backend: `sudo apt install libusb-1.0-0`.

**Good result:** the install finishes without errors.

---

## Step 1. Does the computer see the supply?

```bash
lsusb -d 2a8d:
```

**Good result:** one line containing `Keysight Technologies` (vendor ID `2a8d`).

**Nothing printed:** cable, USB port, or the supply's USB interface is off (front panel:
Utility > I/O Config > USB). Try another cable or port.

---

## Step 2. Can Python list it?

```python
import pyvisa
rm = pyvisa.ResourceManager("@py")
rm.list_resources()
```

**Good result:** a tuple containing something like
`'USB0::0x2A8D::0x1602::MY12345678::INSTR'`. Copy that string; it is the address used below.

**Empty tuple `()` or a permission error:** add a udev rule, then replug the USB cable.

```bash
echo 'SUBSYSTEM=="usb", ATTR{idVendor}=="2a8d", MODE="0666"' | sudo tee /etc/udev/rules.d/99-keysight.rules
sudo udevadm control --reload-rules
```

---

## Step 3. Open it and ask who it is

```python
addr = "USB0::0x2A8D::0x1602::MY12345678::INSTR"   # paste your own address
psu = rm.open_resource(addr)
psu.timeout = 5000
psu.read_termination = "\n"
psu.write_termination = "\n"
psu.query("*IDN?")
```

**Good result:** `'Keysight Technologies,E36311A,MY...,x.xx-x.xx-x.xx'`.

**Timeout:** run `psu.close()` and retry once; if it keeps failing, unplug and replug USB.

---

## Step 4. Read the current state (still read-only)

Check the error queue first, it should be empty:

```python
psu.query("SYST:ERR?")        # expect '+0,"No error"'
```

Output state and set values of channel 1:

```python
psu.query("OUTP? (@1)")       # '0' = off, '1' = on
psu.query("VOLT? (@1)")       # programmed voltage
psu.query("CURR? (@1)")       # programmed current limit
```

Measured values:

```python
psu.query("MEAS:VOLT? (@1)")
psu.query("MEAS:CURR? (@1)")
```

Maximum the channel accepts (use this instead of trusting a table):

```python
psu.query("VOLT? MAX,(@1)")
psu.query("CURR? MAX,(@1)")
```

Repeat with `(@2)` and `(@3)`.

**Good result:** numbers come back, errors stay at `+0`. If a query returns an error
text, copy it and stop here.

---

## Step 5. Change the settings, output stays OFF

Before this step, make sure nothing important is connected to the channel, or that the
values below are safe for it. Use channel 1 and a small value:

```python
psu.write("CURR 0.5,(@1)")        # current limit 0.5 A
psu.write("VOLT 1.0,(@1)")        # 1.0 V
psu.query("SYST:ERR?")            # expect '+0,"No error"'
psu.query("VOLT? (@1)")           # expect about '1.0'
psu.query("CURR? (@1)")           # expect about '0.5'
psu.query("OUTP? (@1)")           # still '0'
```

**Good result:** values read back as written, output is still off.

**An error such as "Data out of range":** the value is above the channel maximum from
step 4. Pick a smaller value.

---

## Step 6. Turn one channel on and off

Ideally with no load, or a load that can take 1 V / 0.5 A.

```python
psu.write("OUTP ON,(@1)")
import time; time.sleep(0.5)
psu.query("MEAS:VOLT? (@1)")      # expect about 1.0
psu.query("MEAS:CURR? (@1)")      # small, near 0 with no load
psu.write("OUTP OFF,(@1)")
psu.query("OUTP? (@1)")           # '0'
```

**Good result:** the front-panel channel indicator lights up and the measured voltage
follows the setting, then goes off again.

---

## Step 7. Close the connection

```python
psu.close()
rm.close()
```

Do not skip it. A connection left open can make the next attempt time out.

---

## Next: the other channels, then several supplies

1. Repeat steps 4-6 for channels 2 and 3 (they have a different maximum than channel 1).
2. Only after all three channels behave, use `e36311a.py`: it wraps exactly these
   commands (`scan`, `status`, `set`, `on`, `off`). Every command in it is one of the
   queries and writes above, so if something fails you can repeat that single step by hand.
3. For several supplies, each one has its own address from step 2 (the serial number
   `MY...` is different). Open each with its own `open_resource` call.

## Quick troubleshooting

| Symptom | Likely cause | What to do |
|---|---|---|
| `lsusb` shows nothing | cable, port, USB interface off | check front-panel I/O settings, other cable |
| `list_resources()` returns `()` | permissions or missing libusb | udev rule (step 2), `sudo apt install libusb-1.0-0` |
| `*IDN?` times out | stale connection | `psu.close()`, replug USB, retry |
| error after a `write` | value outside range or bad syntax | `psu.query("SYST:ERR?")` and read the message |
| value reads back different | rounding or limit | compare with step 4 maximum |
