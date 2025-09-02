# Level0 -> 1

In this level, you start as the `level0` user. The goal is to move to `level1` by analyzing and interacting with the `level0` executable.

---

## Step 1: Explore the Environment

When you log in and list the directory:

```bash
ls
level0
```

The only file present is the executable `level0`. There are no scripts or obvious hints, so we need to analyze the binary itself.

---

## Step 2: Analyze the Binary with GDB

Since there is no source code, **GDB** is the best tool to understand what the binary expects.

```bash
gdb ./level0
```

Set a breakpoint at `main`:

```gdb
b main
run
```

This allows you to pause execution right at the start.

---

## Step 3: Inspect Program Execution

Step through the instructions using `stepi` or `nexti` until you see a key comparison:

```
0x8048ed9 <main+25>     cmp    $0x1a7,%eax
```

**Analysis:**

* The instruction compares the value in the `eax` register to `0x1a7`.
* `0x1a7` in decimal is **423**.
* `%eax` usually contains the numeric value of the first command-line argument.

✅ This indicates the program expects **423** as input.

---

## Step 4: Run the Program with the Correct Input

With the correct input identified, run:

```bash
./level0 423
```

Check which user you are now:

```bash
whoami
```

Output:

```
level1
```

You have successfully escalated to **level1**.

---

## Step 5: Retrieve the Next Level Password

The password for `level1` is stored in `.pass` inside the level1 home directory. You can read it with:

```bash
cat /home/user/level1/.pass
1fe8a524fa4bec01ca4ea2a869af2a02260d4a7d5fe7e7c24d8617e6dca12d3a
```

This will display a hash, which is the credential for the next level.