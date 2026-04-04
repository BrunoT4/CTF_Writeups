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

```bash
>>> from scipy.io import wavfile
>>> rate, data = wavfile.read('main.wav')
>>> data = data.astype(str)
>>> print(" ".join(data))
2002 2507 2001 1508 2006 8500...
```

This is very interesting because the soundbyte is incredibly uniform. In fact, every single frequency seems to be a multiple of 500 if we ignore the least significant digit in decimal. Instinctively my mind is still thinking about LSB encoding, and so I'm looking at these hanging numbers (2, 7, 1, ...), however the fact each value is close to a multiple of 500 tells me that this might be relevant to the actual encoding method. So I decided to divide each value in this array by 500.

```bash
>>> m_data = [x // 500 for x in data]
>>> m_data = m_data.astype(str)
>>> print(list(map(int, m_data)))
[4, 5, 4, 3, 4, 17, 9, 7, 9, 5, 9, ... 12]
```

## Figuring Out the Encoding

While observing the previous list, and running an additional sort, I noticed something very interesting. The range of the numbers is 16, so this very well may be a base16 encoding with some additional padding. I decided to run an additional transformation by lowering the each value by the lowest value (2), which allows us to get a familiar range of 0-15. I also translated each value to its corresponding hex digit and joined the list. This gives a nice clean hex string.

```bash
>>> m_data = [(x // 500) - 2 for x in data]
hex_list = [hex(i)[2:].upper() for i in m_data]
print(''.join(hex_list))
```

All of that gives me this string: 

`23212F7573722F62696E2F656E7620707974686F6E330A696D706F7274206E756D7079206173206E700A66726F6D2073636970792E696F2E77617666696C6520696D706F72742077726974650A66726F6D2062696E617363696920696D706F7274206865786C6966790A66726F6D2072616E646F6D20696D706F72742072616E646F6D0A0A77697468206F70656E282767656E65726174655F7761762E7079272C20277262272920617320663A0A09636F6E74656E74203D20662E7265616428290A09662E636C6F736528290A0A2320436F6E7665727420746869732070726F6772616D20696E746F20616E206172726179206F66206865782076616C7565730A6865785F7374756666203D20286C697374286865786C69667928636F6E74656E74292E6465636F646528227574662D38222929290A0A23204C6F6F70207468726F756768207468652065616368206368617261637465722C20616E6420636F6E76657274207468652068657820612D66206368617261637465727320746F2031302D31350A666F72206920696E2072616E6765286C656E286865785F737475666629293A0A096966206865785F73747566665B695D203D3D202761273A0A09096865785F73747566665B695D203D2031300A09656C6966206865785F73747566665B695D203D3D202762273A0A09096865785F73747566665B695D203D2031310A09656C6966206865785F73747566665B695D203D3D202763273A0A09096865785F73747566665B695D203D2031320A09656C6966206865785F73747566665B695D203D3D202764273A0A09096865785F73747566665B695D203D2031330A09656C6966206865785F73747566665B695D203D3D202765273A0A09096865785F73747566665B695D203D2031340A09656C6966206865785F73747566665B695D203D3D202766273A0A09096865785F73747566665B695D203D2031350A0A092320546F206D616B65207468652070726F6772616D2061637475616C6C792061756469626C652C2031303020686572747A2069732061646465642066726F6D2074686520626567696E6E696E672C207468656E20746865206E756D626572206973206D756C7469706C6965642062790A09232035303020686572747A0A092320506C7573206120636865656B792072616E646F6D20616D6F756E74206F66206E6F6973650A096865785F73747566665B695D203D2031303030202B20696E74286865785F73747566665B695D29202A20353030202B20283130202A2072616E646F6D2829290A0A0A64656620736F756E645F67656E65726174696F6E286E616D652C2072616E645F686578293A0A0923205468652068657820617272617920697320636F6E76657274656420746F20612031362062697420696E74656765722061727261790A097363616C6564203D206E702E696E743136286E702E6172726179286865785F737475666629290A092320536369205069207468656E2077726974657320746865206E756D707920617272617920696E746F2061207761762066696C650A097772697465286E616D652C206C656E286865785F7374756666292C207363616C6564290A0972616E646F6D6E657373203D2072616E645F6865780A0A0A232050756D7020757020746865206D75736963210A23207072696E74282247656E65726174696E67206D61696E2E7761762E2E2E22290A2320736F756E645F67656E65726174696F6E28276D61696E2E77617627290A23207072696E74282247656E65726174696F6E20636F6D706C6574652122290A0A2320596F757220656172732068617665206265656E20626C657373656423207069636F4354467B6D553231435F31735F313333375F35646236623835657D0A`

That when converting to ASCII gave me:

```python
#!/usr/bin/env python3
import numpy as np
from scipy.io.wavfile import write
from binascii import hexlify
from random import random

with open('generate_wav.py', 'rb') as f:
	content = f.read()
	f.close()

# Convert this program into an array of hex values
hex_stuff = (list(hexlify(content).decode("utf-8")))

# Loop through the each character, and convert the hex a-f characters to 10-15
for i in range(len(hex_stuff)):
	if hex_stuff[i] == 'a':
		hex_stuff[i] = 10
	elif hex_stuff[i] == 'b':
		hex_stuff[i] = 11
	elif hex_stuff[i] == 'c':
		hex_stuff[i] = 12
	elif hex_stuff[i] == 'd':
		hex_stuff[i] = 13
	elif hex_stuff[i] == 'e':
		hex_stuff[i] = 14
	elif hex_stuff[i] == 'f':
		hex_stuff[i] = 15

	# To make the program actually audible, 100 hertz is added from the beginning, then the number is multiplied by
	# 500 hertz
	# Plus a cheeky random amount of noise
	hex_stuff[i] = 1000 + int(hex_stuff[i]) * 500 + (10 * random())


def sound_generation(name, rand_hex):
	# The hex array is converted to a 16 bit integer array
	scaled = np.int16(np.array(hex_stuff))
	# Sci Pi then writes the numpy array into a wav file
	write(name, len(hex_stuff), scaled)
	randomness = rand_hex


# Pump up the music!
# print("Generating main.wav...")
# sound_generation('main.wav')
# print("Generation complete!")

# Your ears have been blessed# picoCTF{mU21C_1s_1337_5db6b85e}
```

A python  script!!! Looks like this was the script used to generate the wav using itself, and at the bottom we have our flag.

**Flag:** `picoCTF{mU21C_1s_1337_5db6b85e}`
