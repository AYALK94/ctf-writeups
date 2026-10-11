# Hidden Cipher 2 — CTF Write-up

**Category:** Reverse Engineering  
**Difficulty:** Medium  
**Challenge:** Hidden Cipher 2  
**Service:** `xebec.cylabacademy.net:26947`

> This write-up documents the decoding approach used to recover the flag from the challenge's encoded numeric output. Only interact with the challenge service when you are authorized to do so.

## 1. Challenge Overview

The challenge provides a network service that asks a math question and then outputs a sequence of numbers representing an encoded flag.

The key observation is that the numbers are not arbitrary: each character's ASCII value has been multiplied by the answer to the math question. Once the multiplier is known, the original characters can be recovered by dividing each number by that value and converting the result back to an ASCII character.

## 2. Connect to the Challenge

Use Netcat to connect to the challenge service:

```bash
nc xebec.cylabacademy.net 26947
```

Read the prompt and solve the math question. Keep the answer, because the challenge uses it as the encoding multiplier.

For this solve, the multiplier used to decode the provided output was `10`.

## 3. Understand the Encoding

ASCII assigns numeric values to characters. For example:

| Character | ASCII value |
|---|---:|
| `A` | 65 |
| `a` | 97 |
| `c` | 99 |
| `{` | 123 |

The challenge's encoding can be represented as:

```text
encoded_value = ASCII_value × multiplier
```

To reverse it:

```text
ASCII_value = encoded_value ÷ multiplier
```

For example, if the original character is `a`:

```text
ASCII value of 'a' = 97
97 × 10 = 970
970 ÷ 10 = 97
ASCII 97 = 'a'
```

So the encoded number `970` decodes to `a` when the multiplier is `10`.

## 4. Decode the Numbers with Python

Save the following as `solve.py` or run it in a Python interpreter:

```python
encoded = [
    970, 990, 970, 1000, 1010, 1090, 1210, 1230,
    1090, 520, 1160, 1040, 950, 980, 510, 1040,
    490, 1100, 1000, 950, 990, 490, 1120, 1040,
    510, 1140, 950, 480, 560, 510, 480, 480,
    990, 560, 1000, 1250
]

multiplier = 10
decoded = ""

for value in encoded:
    ascii_value = value // multiplier
    decoded += chr(ascii_value)

print(decoded)
```

Expected output:

```text
academy{m4th_b3h1nd_c1ph3r_08300c8d}
```

### How the code works

1. `encoded` stores the numeric values received from the challenge.
2. `multiplier` stores the answer to the math question for this solve.
3. The `for` loop processes each number in order.
4. `value // multiplier` recovers the ASCII value. Integer division is used because the values divide evenly in this example.
5. `chr(ascii_value)` converts the numeric ASCII value into a character.
6. Each character is appended to `decoded`, reconstructing the flag.

For a more defensive decoder, check that every number is divisible by the multiplier before converting it:

```python
for value in encoded:
    if value % multiplier != 0:
        raise ValueError(f"{value} is not divisible by {multiplier}")
    decoded += chr(value // multiplier)
```

## 5. Flag

```text
academy{m4th_b3h1nd_c1ph3r_08300c8d}
```

## 6. Key Takeaways

- **ASCII** maps characters to numeric values.
- **Encoding** transforms data into another representation; it does not necessarily provide security.
- A reversible transformation can often be undone by identifying its rule and reversing the operation.
- In this challenge, multiplication by a known value is reversed using division.
- In a real solve, confirm the multiplier from the math prompt instead of assuming it. The multiplier can be session-specific.

## 7. Security Note: Encoding Is Not Encryption

Multiplying ASCII values by a small number is an obfuscation technique, not secure encryption. Anyone who identifies the transformation and the multiplier can recover the original text.

Real systems should use established cryptographic algorithms and sound key management to protect sensitive information. Passwords should generally be stored using a dedicated password-hashing scheme, not reversible encoding.

---

**Challenge solved:** Hidden Cipher 2  
**Recovered flag:** `academy{m4th_b3h1nd_c1ph3r_08300c8d}`
