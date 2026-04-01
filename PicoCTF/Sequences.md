# picoCTF 2022 — Sequences

**Category:** Cryptography, **Difficulty:** Hard

---

## Author Notes

This is an interesting problem because it's really just a math optimization problem, as opposed to your traditional cryptanalysis challenge that you'd see in a ctf. Nonetheless I enjoy math and it seemed pretty straight forward, so I thought I'd make a quick write up here.

---

## Challenge Overview

The challenge gives this Python code:

```python
import math
import hashlib
import sys
from tqdm import tqdm
import functools

ITERS = int(2e7)
VERIF_KEY = "96cc5f3b460732b442814fd33cf8537c"
ENCRYPTED_FLAG = bytes.fromhex("42cbbce1487b443de1acf4834baed794f4bbd0dfe2d6046e248ff7962b")

# This will overflow the stack, it will need to be significantly optimized in order to get the answer :)
@functools.cache
def m_func(i):
    if i == 0: return 1
    if i == 1: return 2
    if i == 2: return 3
    if i == 3: return 4

    return 55692*m_func(i-4) - 9549*m_func(i-3) + 301*m_func(i-2) + 21*m_func(i-1)


# Decrypt the flag
def decrypt_flag(sol):
    sol = sol % (10**10000)
    sol = str(sol)
    sol_md5 = hashlib.md5(sol.encode()).hexdigest()

    if sol_md5 != VERIF_KEY:
        print("Incorrect solution")
        sys.exit(1)

    key = hashlib.sha256(sol.encode()).digest()
    flag = bytearray([char ^ key[i] for i, char in enumerate(ENCRYPTED_FLAG)]).decode()

    print(flag)

if __name__ == "__main__":
    sol = m_func(ITERS)
    decrypt_flag(sol)
```

The challenge involes optimizing a recursive function so that it does not overflow and take a billion years to run (yes I am being serious). The result of the function will be the decryption key used to decrypt the ciphertext and output the flag.

---

## Step 1: Rewrite the recurrence

The recursive function defines the sequence

\[
a_0 = 1,\quad a_1 = 2,\quad a_2 = 3,\quad a_3 = 4
\]

and for \(n \ge 4\),

\[
a_n = 55692a_{n-4} - 9549a_{n-3} + 301a_{n-2} + 21a_{n-1}.
\]

I rewrote it in a more standard order as

\[
a_n = 21a_{n-1} + 301a_{n-2} - 9549a_{n-3} + 55692a_{n-4}.
\]

This makes it clear that it is just a linear recurrence of order 4.

---

## Step 2: Key observation about the modulus

The biggest thing that simplifies the challenge is in `decrypt_flag`:

```python
sol = sol % (10**10000)
```

That means I do **not** need the full value of \(a_n\). I only need

\[
a_n \bmod 10^{10000}.
\]

That is huge because it means I can do the entire recurrence modulo

\[
M = 10^{10000}
\]

from the start.

Since linear recurrences behave nicely under modular arithmetic, this loses no information for the final answer. It just keeps the numbers manageable.

So the actual goal becomes:

\[
a_{20000000} \bmod 10^{10000}.
\]

---

## Step 3: Convert the recurrence into a matrix

I define the state vector

\[
\mathbf{x}_n =
\begin{bmatrix}
a_n \\
a_{n-1} \\
a_{n-2} \\
a_{n-3}
\end{bmatrix}.
\]

Then the recurrence can be written as

\[
\mathbf{x}_n = A\mathbf{x}_{n-1}
\]

where

\[
A =
\begin{bmatrix}
21 & 301 & -9549 & 55692 \\
1 & 0 & 0 & 0 \\
0 & 1 & 0 & 0 \\
0 & 0 & 1 & 0
\end{bmatrix}.
\]

So instead of recursively computing one term at a time, I can compute

\[
\mathbf{x}_n = A^{n-3}\mathbf{x}_3
\]

with

\[
\mathbf{x}_3 =
\begin{bmatrix}
4 \\
3 \\
2 \\
1
\end{bmatrix}.
\]

The answer I want is the first entry of that vector.

This is much better because I can compute \(A^{n-3}\) using fast exponentiation, which only takes \(O(\log n)\) multiplications.

---

## Step 4: Adding the Optimization to the Program

The optimized solution is to compute the matrix power modulo \(10^{10000}\) and then feed the result into the original decryption logic.

This is the modification I made:

```python
import hashlib
import sys

sys.set_int_max_str_digits(0)

ITERS = int(2e7)
VERIF_KEY = "96cc5f3b460732b442814fd33cf8537c"
ENCRYPTED_FLAG = bytes.fromhex("42cbbce1487b443de1acf4834baed794f4bbd0dfe2d6046e248ff7962b")


# START OF MODIFIED CODE

MOD = 10**10000

def mat_mul(A, B, mod):
    n = len(A)
    m = len(B[0])
    kdim = len(B)
    out = [[0] * m for _ in range(n)]

    for i in range(n):
        for k in range(kdim):
            if A[i][k] == 0:
                continue
            for j in range(m):
                out[i][j] = (out[i][j] + A[i][k] * B[k][j]) % mod
    return out

def mat_pow(A, e, mod):
    n = len(A)
    result = [[1 if i == j else 0 for j in range(n)] for i in range(n)]

    while e > 0:
        if e & 1:
            result = mat_mul(result, A, mod)
        A = mat_mul(A, A, mod)
        e >>= 1

    return result

def mat_vec_mul(A, v, mod):
    out = [0] * len(A)
    for i in range(len(A)):
        total = 0
        for j in range(len(v)):
            total += A[i][j] * v[j]
        out[i] = total % mod
    return out

def m_func(i):
    if i == 0:
        return 1
    if i == 1:
        return 2
    if i == 2:
        return 3
    if i == 3:
        return 4

    A = [
        [21 % MOD, 301 % MOD, (-9549) % MOD, 55692 % MOD],
        [1, 0, 0, 0],
        [0, 1, 0, 0],
        [0, 0, 1, 0],
    ]

    x3 = [4, 3, 2, 1]
    P = mat_pow(A, i - 3, MOD)
    xi = mat_vec_mul(P, x3, MOD)
    return xi[0]

# END OF MODIFIED CODE

def decrypt_flag(sol):
    sol %= MOD
    sol_str = str(sol)
    sol_md5 = hashlib.md5(sol_str.encode()).hexdigest()

    if sol_md5 != VERIF_KEY:
        print("Incorrect solution")
        sys.exit(1)

    key = hashlib.sha256(sol_str.encode()).digest()
    flag = bytearray([char ^ key[i] for i, char in enumerate(ENCRYPTED_FLAG)]).decode()
    print(flag)

if __name__ == "__main__":
    sol = m_func(ITERS)
    decrypt_flag(sol)
```

---


This optimized version uses:
- modular arithmetic to keep the numbers bounded
- matrix exponentiation to reduce the work to logarithmic time

So instead of trying to build up all 20,000,000 terms, I just exponentiate a 4x4 matrix with repeated squaring.

---

## Flag

After about 10 seconds of running the optimized solver gives us the flag:

**Flag:** `picoCTF{b1g_numb3rs_689693c6}`
