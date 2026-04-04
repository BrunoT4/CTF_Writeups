# picoCTF 2021 — Surfing the Waves

**Category:** Digital Forensics, **Difficulty:** Hard

## Description

`While you're going through the FBI's servers, you stumble across their incredible taste in music. One you found is particularly interesting, see if you can find the flag!` [main.wav](https://challenge-files.picoctf.net/c_wily_courier/6d28b9a19ca845d146bb5060dedfe2dbc61f412596ad6aafb5736023ae701933/main.wav)

---

## Challenge Overview

Main.wav is a very short (~1 second) audio clip of what appears to be random noise. At first glance, it seems to be a very basic audio steganography problem, nothing that an existing library or web app can't solve. However the challenge quickly proves to be much more intricate, as it seems the file may use a custom encoding created by the challenge authors.

---

## Initial Steganography

My intial assumption was that this file uses some existing form of audio steganography (most likely LSB encoding), which makes sense given this is a digital forensics problem. I began by plugging it into a couple online tools I found: https://futureboy.us/stegano/decinput.html, https://jdecime.com/ctf_audio, https://raffertysmith.com/steganography. All of these gave me nothing. Just a whole bunch of errors and warning messages, and one even suggested that the file might require a password to decode. As frustrating as that sounded I knew it couldn't be the case, because with no additional information I wouldn't even know how to go about retrieving that. After a sufficient amount of walking in a circle I decided to actually look at the audio data myself.

## A Very Interesting Sound Wave

I wanted a way to convert the wav into numerical data (an array of each frequency in the sound wave), I.e a Fourier Transform. Luckily I know a python library that can do exactly this: scipy. After doing a quick little installation using brew, I got to inspecting. I used the following inline scripting:


