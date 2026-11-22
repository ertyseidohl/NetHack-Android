This document was written by Gemini 3.6 flash. It may contain errors or hallucinations.

# 64-Bit Compatibility & NetHack Engine Migration Plan

This document outlines the identified 64-bit architecture incompatibilities across the NetHack C engine, JNI bridge (`winandroid.c`), and file serialization format, along with the plan for auditing and resolving them.

---

## Category 1: Fixed-Width Struct Headers & Binary File Compatibility

* **Issue**: `struct version_info` and `struct savefile_info` in `include/global.h` originally used native `unsigned long` fields (4 bytes on 32-bit ILP32 vs 8 bytes on 64-bit LP64). This caused binary file header size mismatches (20 bytes vs 40 bytes) when reading level/dungeon data files (`dungeon`, `*.lev`) and save files on 64-bit architectures (`x86_64`, `arm64-v8a`).
* **Affected Files**: [`include/global.h`](file:///home/erty/workspace/NetHack-Android/include/global.h), [`src/version.c`](file:///home/erty/workspace/NetHack-Android/src/version.c), [`src/dungeon.c`](file:///home/erty/workspace/NetHack-Android/src/dungeon.c), [`src/files.c`](file:///home/erty/workspace/NetHack-Android/src/files.c), [`util/makedefs.c`](file:///home/erty/workspace/NetHack-Android/util/makedefs.c)
* **Action Plan**:
    - Replace `unsigned long` in `struct version_info` and `struct savefile_info` with fixed 32-bit `uint32` (from [`include/integer.h`](file:///home/erty/workspace/NetHack-Android/include/integer.h)). Note: Ensure correct spelling (`uint32` instead of `unit32`).
    - Recompile host utilities (`dgn_comp`, `lev_comp`, `makedefs`), regenerate all level data files (`dungeon`, `*.lev`), and copy updated assets into `sys/android/app/assets/nethackdir/`.

---

## Category 2: Menu Identifier Truncation via `anything` Union

* **Issue**: In [`sys/android/winandroid.c`](file:///home/erty/workspace/NetHack-Android/sys/android/winandroid.c), object and window menu identifiers stored in NetHack's `anything` union (`ident->a_obj`, `ident->a_void`) are 64-bit pointers (8 bytes). Casting them to/from `(int)` or `ident->a_int` (4 bytes) when communicating across JNI truncates 64-bit pointers, causing a `SIGSEGV` crash when dereferenced by the C engine.
* **Affected Files**: [`sys/android/winandroid.c`](file:///home/erty/workspace/NetHack-Android/sys/android/winandroid.c), `com.tbd.forkfront.NetHackIO`
* **Action Plan**:
    - Audit all `ident` handling in `and_add_menu`, `and_select_menu`, and JNI callbacks.
    - Pass menu identifiers across JNI using `jlong` (64-bit integer) and map to `a_void` or `a_ulong`.

---

## Category 3: Pointer Subtraction & Offset Casting to `int`

* **Issue**: Casting pointer differences or string search returns (`index()`) to `(int)` (32-bit signed integer) rather than `ptrdiff_t` or direct pointer arithmetic. On 64-bit Android platforms where heap memory is allocated above 2 GB/4 GB (`0x7fff...`), casting pointers to `(int)` causes integer overflow into negative values.
* **Affected Files**: [`sys/android/winandroid.c`](file:///home/erty/workspace/NetHack-Android/sys/android/winandroid.c) (e.g., `(int)index(choices, '\033') - (int)choices`), string/path parsing routines
* **Action Plan**:
    - Replace `(int)ptr2 - (int)ptr1` expressions with standard pointer arithmetic `(ptr2 - ptr1)`.
    - Replace `(int)` pointer casts with `uintptr_t` or `ptrdiff_t` where offset calculation is required.

---

## Category 4: Binary File Serialization (`bread` / `bwrite`) of Native Types

* **Issue**: Engine file I/O operations (`bwrite`/`bread`) directly read and write native C types like `unsigned long` or pointers to save files and level files. Size differences between 32-bit and 64-bit ABIs corrupt binary state files when saved on one architecture and restored on another.
* **Affected Files**: [`src/files.c`](file:///home/erty/workspace/NetHack-Android/src/files.c), [`src/save.c`](file:///home/erty/workspace/NetHack-Android/src/save.c), [`src/restore.c`](file:///home/erty/workspace/NetHack-Android/src/restore.c)
* **Action Plan**:
    - Audit `bwrite()` and `bread()` calls across save/restore routines.
    - Convert file serialization buffers to fixed-width integer types (`int32_t`, `uint32_t`).

---

## Category 5: Bit-Shift Masking and Integer Promotion on `long` vs `uint32`

* **Issue**: Bitwise shift operations using `(1L << 31)` or `(1UL << 32)` behave differently across 32-bit and 64-bit architectures. Shifting 32 bits on a 32-bit `unsigned long` is undefined behavior in C, while `1L << 31` sign-extends to 64-bit negative values on 64-bit targets.
* **Affected Files**: [`include/config.h`](file:///home/erty/workspace/NetHack-Android/include/config.h), [`include/flag.h`](file:///home/erty/workspace/NetHack-Android/include/flag.h), [`src/decl.c`](file:///home/erty/workspace/NetHack-Android/src/decl.c), [`src/options.c`](file:///home/erty/workspace/NetHack-Android/src/options.c)
* **Action Plan**:
    - Audit bitmask flags and macro definitions across engine headers.
    - Enforce explicit `uint32` casts for bit shifts (e.g., `(uint32)1 << n`).