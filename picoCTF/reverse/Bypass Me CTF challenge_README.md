# CTF Write-Up: Bypass Me

> **Category:** Reverse Engineering  
> **Difficulty:** Medium  
> **Challenge:** Bypass Me  
> **Platform:** Academy CTF  
> **Target file:** `bypassme.bin`  
> **Status:** Solved

## Overview

The goal of this challenge was to inspect a password-protected executable, understand how it handled user input and authentication, and retrieve the hidden flag.

Rather than guessing repeatedly, I used basic binary inspection to identify useful strings and function names. I then ran the program, tested the password, and captured the flag displayed after successful authentication.

## Tools Used

- Linux terminal
- `strings`
- `bypassme.bin`
- Optional tools for deeper analysis: `gdb`, `objdump`, `readelf`, and `file`

> **Environment note:** The challenge shell did not have `file` or `checksec` installed. This did not prevent the initial analysis.

## Step 1: Inspect the Binary

I first tried to identify the executable type:

```bash
file bypassme.bin
```

The command was unavailable in the provided environment:

```text
-bash: file: command not found
```

I also checked security properties with:

```bash
checksec --file=bypassme.bin
```

However, `checksec` was not installed either. I continued with the available tools.

## Step 2: Extract Readable Strings

I ran:

```bash
strings -n 4 bypassme.bin
```

The output exposed several useful strings and function names, including:

```text
isalpha
strcspn
strcmp
fopen
decode_password
sanitize
auth_sequence
Raw Input:      [%s]
Sanitized Input:[%s]
Hint: Input must match something special...
../../root/flag.txt
Flag: %s
Access Denied
All attempts used. Try harder!
```

### What These Clues Suggest

- `decode_password` appears to be related to preparing or decoding the expected password.
- `sanitize` suggests that the program processes user input before authentication.
- `strcmp` is a C library function commonly used to compare strings.
- `../../root/flag.txt` indicates the path the program attempts to open for the flag.
- The raw and sanitized input messages make it possible to observe how the program handles the submitted password.

These names provide useful leads, but the exact implementation of each function would need to be confirmed by examining the source or disassembly.

## Step 3: Understand the Likely Authentication Flow

Based on the strings and function names, the program appears to follow a flow similar to this:

1. Read a password from the user.
2. Sanitize the input.
3. Prepare or decode the expected password.
4. Compare the values.
5. Display an access-denied message if authentication fails.
6. Read and display the flag if authentication succeeds.

This is an inferred flow based on the available output, not a substitute for a full source-code or disassembly analysis.

## Step 4: Run the Program

I executed the binary:

```bash
./bypassme.bin
```

The program displayed its banner and requested a password:

```text
[3 tries left] Enter password:
```

I entered the password:

```text
SuperSecure
```

The program echoed the input and showed that the sanitized value was unchanged:

```text
Raw Input:      [SuperSecure]
Sanitized Input:[SuperSecure]
```

Authentication then succeeded, and the program printed the flag.

## Step 5: Capture the Flag

**Flag:**

```text
academy{d3bugg3r_p0w3r_is_4w3s0m3_e85bfcda}
```

The flag was printed by the program after successful authentication.

## Lessons Learned

- `strings` is a useful first step when investigating an unfamiliar executable.
- Function names such as `sanitize`, `decode_password`, and `strcmp` can reveal where to focus further analysis.
- A missing utility such as `file` or `checksec` does not necessarily stop a binary-analysis workflow.
- Debuggers such as GDB can help inspect functions and runtime behavior when available.
- Sanitization and authentication are separate concepts: sanitization transforms or filters input, while authentication determines whether the input is accepted.
- In a real security assessment, findings should be verified and documented rather than inferred from symbol names alone.

## Further Analysis (Optional)

If GDB is available in the challenge environment, inspect the relevant functions:

```bash
gdb -q ./bypassme.bin
```

Inside GDB:

```text
(gdb) info functions decode_password
(gdb) disassemble decode_password
(gdb) disassemble sanitize
```

You can also check whether other analysis tools are available:

```bash
command -v gdb
command -v objdump
command -v readelf
command -v strings
```

These commands can help investigate the implementation in more depth. Use them only against binaries you are authorized to analyze.

## Conclusion

I solved **Bypass Me** by inspecting the executable's readable strings, identifying promising function names, running the binary, and verifying that the password `SuperSecure` allowed authentication to succeed. The program then revealed the challenge flag.

---

*Write-up based on my challenge session. The password and flag above are the values observed during that run.*
