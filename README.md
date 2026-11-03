# 🔌 lab-instruments

Small, dependable Python tools for controlling bench instruments over **SCPI**, built for
detector test setups where a wrong click on a power supply can cost a chip.

![Python](https://img.shields.io/badge/python-3.8%2B-blue)
![PyVISA](https://img.shields.io/badge/backend-PyVISA-informational)
![Status](https://img.shields.io/badge/status-bring--up-orange)

> **One rule:** read first, change second. Every tool here can inspect an instrument
> without touching its settings, and anything that changes state says so.

---

## ✨ What's here

| Instrument | Interface | Folder | Status |
|---|---|---|---|
| Keysight **E36311A** triple-output DC supply | USB (SCPI) | [`keysight/`](keysight/) | 🚧 bring-up in progress |

More instruments (scope, source-meter, temperature control, ...) will get their own
folder as they are added.

---

## 🚀 Quick start

Works with **Python 3.8** (no upgrade needed).

```bash
git clone https://github.com/huangx-git/lab-instruments.git
cd lab-instruments
pip install -r requirements.txt
```

Then follow the step-by-step guide for your instrument, for example the supply:

👉 [`keysight/E36311A_BRINGUP.md`](keysight/E36311A_BRINGUP.md)

It walks through eight small steps, from "does the computer see it?" to switching a
channel on and off. The first four steps only read.

If `pip` rejects the PyVISA version on Python 3.8: `pip install "pyvisa<1.15" pyvisa-py pyusb`.

---

## 🧭 Repository layout

```
lab-instruments/
├── README.md
├── requirements.txt
├── .gitignore
└── keysight/
    ├── E36311A_BRINGUP.md     # step-by-step, run line by line in a Python session
    └── e36311a.py             # the same commands as a small CLI
```

---

## 🛡️ Safety principles

These apply to every tool in this repo.

1. **Read-only by default.** Scanning, identifying and status commands never write.
2. **Setting is not switching on.** Programming a voltage or current limit does not enable
   the output; turning it on is a separate, explicit command.
3. **Ask the instrument for its limits** instead of trusting a table in the code.
4. **Read back after every write**, and stop on a mismatch.
5. **Check the error queue** (`SYST:ERR?`) after commands, so a rejected value is not
   silently ignored.

---

## 🧪 Example: Keysight E36311A

```bash
python keysight/e36311a.py scan             # list USB instruments
python keysight/e36311a.py status           # set and measured values, all channels
python keysight/e36311a.py set 1 5.0 2.0    # CH1: 5.0 V, 2.0 A limit (output unchanged)
python keysight/e36311a.py on 1
python keysight/e36311a.py off 1 2 3
```

> ⚠️ The script has not been run against hardware yet. Use the bring-up guide first and
> check each command by hand on a safe load.

---

## ➕ Adding an instrument

1. Create a folder named after the vendor, e.g. `rigol/`.
2. Add a bring-up guide with the same small read-only steps first.
3. Wrap the commands you verified by hand in a script, keeping the safety principles above.
4. Add a row to the table at the top of this README.

---

## 🗺️ Roadmap

- [ ] Verify `e36311a.py` on hardware
- [ ] Several supplies at once, with named channels (for example `D0_1V2`)
- [ ] Ordered power-up and power-down sequences with current limits
- [ ] Hook into the ETROC Pathfinder test flow

---

## 📝 Notes

- Serial numbers and IP addresses are placeholders in the docs (`MY12345678`); keep real
  ones out of the repository.
- No license has been chosen yet.
