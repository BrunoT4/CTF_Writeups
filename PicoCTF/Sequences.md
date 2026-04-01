# picoCTF 2022 — Sequences (unfinished)

**Category:** Cryptography, **Difficulty:** Hard

---

## Author Notes

This is an interesting problem because it's really just a math optimization problem, as opposed to your traditional cryptanalysis challenge that you'd see in a ctf. Nonetheless I enjoy math and it seemed pretty straight forward, so I thought I'd make a quick write up here.

---

## Challenge Overview

The challenge involes optimizing a recursive function so that it does not overflow and take a billion years to run (yes I am being serious). The result of the function will be the decryption key used to decrypt the ciphertext and output the flag.

---

## My Solution

I represented the recursive function as a matrix and used reductions to transform it into a non-recursive equation. My solution is on paper so hopefully my hand-writing is legible. If not, I do not go to school to study handwriting, so bare with me.
