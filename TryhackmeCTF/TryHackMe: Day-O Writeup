# TryHackMe: Day-O Writeup

**Room:** Day-O (Medium) | **Category:** Memory Forensics, Malware Analysis
**Tools:** Volatility 3, Speakeasy, Python (pycryptodome), base64/iconv, xxd

> Flag and some values are partly redacted to avoid spoiling the room. Try it yourself first.

## Scenario

Sophia Reyes opened a document review package. The preview froze and an unexpected executable region appeared in the application's memory. Given a Windows memory dump (`memory.mem`), find the payload, work out how it runs, and recover what it was set to do.

## Attack chain

```
review-cover.png (payload appended after IEND)
        -> shellcode in private RWX region of ReviewClient.exe (0x8e0000)
        -> kernel32.WinExec("powershell.exe -EncodedCommand ...")
        -> AES-256-CBC decrypt of review-cache.dat
        -> review-cache.zip (release/approval.txt, contacts.csv) -> flag
```

## Step 1: Find the injected code (Q1, Q2)

```bash
vol -s symbols -f memory.mem windows.malfind
```

`malfind` returned many hits. Most were false positives (`MsMpEng.exe`, `SearchApp.exe`, `smartscreen`, `WWAHost`, `backgroundTask`, `LocalBridge`): repeating stubs, CFG structures and hooking trampolines.

The real hit was the only one with real shellcode, in a document app:

```
6260  ReviewClient.e  0x8e0000  0x8e0fff  VadS  PAGE_EXECUTE_READWRITE  Function prologue
```

The code is 32-bit (`push ebp; mov ebp, esp`), reads `fs:[0x30]` (the PEB) and walks the export table to resolve an API by name. It also pushes the stack string `0x456e6957 0x6578 0x63`, which reads `WinExec`.

| Question | Answer |
|---|---|
| Q1: Executable with the payload | `ReviewClient.exe` |
| Q2: Start of private executable region | `0x8e0000` |

## Step 2: Dump and emulate the shellcode (Q5)

```bash
vol -s symbols -f memory.mem windows.malfind --pid 6260 --dump
speakeasy -t pid.6260.vad.0x8e0000-0x8e0fff.dmp -r -a x86
```

Speakeasy output:

```
kernel32.WinExec("powershell.exe -NoProfile -EncodedCommand JABrAD0A...", 0x5)
```

**Q5: `WinExec`**

## Step 3: Decode the PowerShell stage (Q6)

`-EncodedCommand` is Base64 of UTF-16LE text:

```bash
echo '<base64>' | base64 -d | iconv -f UTF-16LE -t UTF-8
```

Decoded:

```powershell
$k=[Convert]::FromBase64String('fXFK...<redacted>...');
$b=[IO.File]::ReadAllBytes('review-cache.dat');
$a=[Security.Cryptography.Aes]::Create();
$a.Key=$k; $a.IV=$b[0..15]; $a.Mode='CBC'; $a.Padding='PKCS7';
$d=$a.CreateDecryptor().TransformFinalBlock($b,16,$b.Length-16);
[IO.File]::WriteAllBytes('review-cache.zip',$d)
```

The stage uses AES-256-CBC. The Base64 key decodes to 32 bytes, the IV is the first 16 bytes of the file, and the rest is ciphertext.

**Q6:** the Base64 string inside `FromBase64String(...)`.

## Step 4: Find the PNG that carries the payload (Q3, Q4)

```bash
vol -s symbols -f memory.mem windows.filescan | grep -i -E "review-cache|\.png"
```

Relevant files, all under `\Users\s.reyes\Downloads\DocumentReview\`:

- `review-cover.png`
- `review-cache.dat` (a copy also sits in `\LabTransfer\`)

Dumped with `windows.dumpfiles --virtaddr <addr>`. I then parsed the PNG chunks and searched for the shellcode's first bytes (`55 89 e5 83 ec 18 31 f6`):

```
IHDR at 8
IDAT at 33 (len 28772)
IEND at 28817 -> chunk ends at byte 28829
shellcode offset: 28829
```

The shellcode starts exactly where `IEND` ends, so it is **appended to a valid PNG**, not hidden in the pixels or a custom chunk. I confirmed the appended bytes are identical to the shellcode dumped from process memory (`True`).

| Question | Answer |
|---|---|
| Q3: File with the same payload | `review-cover.png` |
| Q4: MITRE sub-technique | `T1027.009` Obfuscated Files or Information: Embedded Payloads |

Note: because the data is appended rather than pixel-embedded, I chose Embedded Payloads over `T1027.003` (Steganography). If the room does not accept it, try `T1027.003`.

## Step 5: Decrypt the cache and get the flag (Q7)

```python
import base64
from Crypto.Cipher import AES
key = base64.b64decode('<key from Step 3>')
b = open('dumped/<review-cache dump>','rb').read().rstrip(b'\x00')
b += b'\x00' * (-len(b) % 16)
d = AES.new(key, AES.MODE_CBC, b[:16]).decrypt(b[16:])
# strip PKCS7 padding, write as zip
```

The result starts with `PK\x03\x04`. All three copies of `review-cache.dat` decrypt to the same 499-byte zip. Extracting it gives:

```
release/approval.txt
release/contacts.csv
```

`approval.txt` contains the approval reference, which is the flag: `THM{************************}`

## Answers

| # | Answer |
|---|---|
| Q1 | `ReviewClient.exe` |
| Q2 | `0x8e0000` |
| Q3 | `review-cover.png` |
| Q4 | `T1027.009` |
| Q5 | `WinExec` |
| Q6 | Base64 AES-256 key from the PowerShell stage |
| Q7 | `THM{...}` (redacted) |

## MITRE ATT&CK mapping

- **T1055**: Process Injection (shellcode in an RWX region of a legitimate app)
- **T1027.009**: Obfuscated Files or Information: Embedded Payloads
- **T1059.001**: Command and Scripting Interpreter: PowerShell
- **T1140**: Deobfuscate/Decode Files or Information (AES decryption of the cache)

## Takeaways

- A process name tells only part of the story. Compare what is running with what its memory contains.
- `malfind` is noisy. Triage by looking for real shellcode in an app that should not have it.
- Stack-string pushes and PEB walks (`fs:[0x30]`) are strong shellcode indicators.
- Check where a payload sits in a carrier file (after `IEND`, in a chunk, or in the pixels) before choosing a technique.
- Emulating shellcode with Speakeasy is a fast way to get its API calls and arguments.
