
# picoCTF 2026 — SideChannel

**Category:** Digital Forensics

---

## Author Notes

This is a problem from picoCTF 2022. It was a problem I solved back in high school, but I wanted to revisit it since it was my first introduction to Side Channels. I also used a very monkey patch-esque methodology for solving the problem, so I figured I'd redo it with some automated scripting. I also re-did this on a laptop with an M1 chip, and I didn't expect Docker to be such a hassle to work with for this :(.

---

## Challenge Overview

The challenge gives you a host address and port to connect to, that upon connection asks for a PIN. Variable on wether you enter the correct PIN or not, you either get an "Access Denied" message, or presumably the flag (if you enter the correct PIN). The problem also provides the ELF file for the pin_checker program, which you can download and analyze on your local machine (The correct PIN is the same for both the provided ELF and the version running on saturn.picoctf.net)

---

## Initial Observations

My first course of action was to connect to the server and just see what was on it. Immediately I got hit with the pin checker program asking me for the PIN. Since the program starts itself upon connection, and the machine boots you out after entering a PIN, it's safe to assume not much can be done here. That's okay, because the problem actually provides you with that same pin checker program, so I can just run a local analysis on the file first. I decided to reverse engineer the program on Ghidra to see what information I could get, but the main function seemed to hefty to be decompiled. This makes sense as one of the hints says: "Attempting to reverse-engineer or exploit the binary won't help you." Luckily I was able to retrieve the length of the expected PIN as 8, which will be helpful later. Since I couldn't pull any more information from reversing the binary, I'd need a different approach. 

---

## The Side Channel

Since the program is expecting a PIN (Personal Identification Number), it's a safe assumption that it is only expecting printable digits (0-9). The hint also suggests the user reads about timing-based side channels. A side channel involves observing information of a physical implementation of a system to understand its functionality. This could be anything such as how much power a machine uses to make a computation, or the frequency of an electro-magnetic wave released by a device, or in this case, how much time it takes to check our PIN.
