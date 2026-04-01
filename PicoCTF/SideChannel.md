
# picoCTF 2026 — SideChannel

**Category:** Digital Forensics

---

## Author Notes

This is a problem from picoCTF 2022. It was a problem I solved back in high school, but I wanted to revisit it since it was my first introduction to Side Channels. I also used a very monkey patch-esque methodology for solving the problem, so I figured I'd redo it with some automated scripting. I also re-did this on a laptop with an M1 chip, and I didn't expect Docker to be such a hassle to work with for this :(.

---

## Challenge Overview

The challenge gives you a host address and port to connect to, that upon connection asks for a PIN. Variable on wether you enter the correct PIN or not, you either get an "Access Denied" message, or presumably the flag (if you enter the correct PIN). The problem also provides the ELF file for the pin_checker program, which you can download and analyze on your local machine.

---

## Initial Observations



