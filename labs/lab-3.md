# Lab 3: The Role of the Linker and Loader
---

## วัตถุประสงค์

1. เข้าใจขั้นตอนการแปลง source code (`.c`) เป็น executable
2. ใช้ `gcc -c` แยก compile กับ link ได้
3. อธิบายความแตกต่างระหว่าง static linking กับ dynamic linking
4. ใช้เครื่องมือ `file`, `nm`, `ldd` ตรวจสอบไฟล์แต่ละขั้นตอน

## สิ่งที่ต้องเตรียม

```bash
sudo apt install gcc build-essential -y
mkdir -p ~/lab_linker && cd ~/lab_linker
```

---

## Part 1 — สร้าง Source Code

สร้างไฟล์ 3 ไฟล์:

**helper.h:**
```c
#ifndef HELPER_H
#define HELPER_H
void greet(const char *name);
#endif
```

**helper.c:**
```c
#include <stdio.h>
void greet(const char *name) {
    printf("Hello, %s!\n", name);
}
```

**main.c:**
```c
#include <stdio.h>
#include <math.h>
#include "helper.h"
int main() {
    printf("sqrt(2.0) = %.4f\n", sqrt(2.0));
    greet("Lab Student");
    return 0;
}
```

---

## Part 2 — Compile: สร้าง Object File

### Step 2.1 — Compile แยกเป็น object file

```bash
gcc -c main.c -o main.o
gcc -c helper.c -o helper.o
```

### Step 2.2 — ตรวจสอบไฟล์

```bash
file main.o
file helper.o
```

> **คำถาม 2.1:** `file main.o` แสดงผลว่าอะไร? ไฟล์ชนิดนี้คืออะไร?
>
> ```
> ตอบ:
>
>
> ```

### Step 2.3 — ดู symbol table ด้วย `nm`

```bash
nm main.o
```

> **คำถาม 2.2:** สัญลักษณ์ `U` หน้า `sqrt` และ `greet` หมายถึงอะไร?
>
> **Hint:** U = Undefined — symbol ถูกใช้แต่ยังไม่มี definition ในไฟล์นี้
>
> ```
> ตอบ:
>
>
> ```

---

## Part 3 — Link: สร้าง Executable

### Step 3.1 — ลอง link โดยไม่ใส่ `-lm`

```bash
gcc -o main main.o helper.o
```

> **คำถาม 3.1:** เกิด error อะไร? ทำไม link ไม่สำเร็จ?
>
> ```
> ตอบ:
>
>
> ```

### Step 3.2 — Link ให้สมบูรณ์

```bash
gcc -o main main.o helper.o -lm
file main
```

> **คำถาม 3.2:** เปรียบเทียบ `file main.o` กับ `file main` — ต่างกันอย่างไร?
>
> ```
> ตอบ:
>
>
> ```

---

## Part 4 — Dynamic vs Static Linking

### Step 4.1 — ดู dynamic dependencies

```bash
ldd main
```

> **คำถาม 4.1:** `ldd` แสดง library อะไรบ้าง? `libm.so` และ `libc.so` คืออะไร?
>
> ```
> ตอบ:
>
>
> ```

### Step 4.2 — เปรียบเทียบ Static กับ Dynamic

```bash
gcc -o main_static main.o helper.o -lm -static
ls -la main main_static
ldd main_static
```

> **คำถาม 4.2:** ขนาดของ `main` vs `main_static` ต่างกันแค่ไหน? ทำไม static ถึงใหญ่กว่า?
>
> ```
> ตอบ:
>
>
> ```

### Step 4.3 — รันทั้งสองแบบ

```bash
./main
./main_static
```

> **คำถาม 4.3:** ผลลัพธ์เหมือนกันไหม? อธิบายว่าทำไม
>
> ```
> ตอบ:
>
>
> ```

---

## Part 5 — สรุป

### ขั้นตอนทั้งหมด

```
  .c  ──compile──►  .o  ──link──►  executable  ──load──►  process
       (gcc -c)          (gcc -o)                (./main)
```

| ขั้นตอน | Input | Output | คำสั่ง | ตรวจสอบด้วย |
|---------|-------|--------|--------|------------|
| Compile | `.c` | `.o` | `gcc -c main.c` | `file`, `nm` |
| Link | `.o` + libs | executable | `gcc -o main *.o -lm` | `file`, `nm` |
| Load & Run | executable | process | `./main` | `ldd` |

ปกติเราใช้ `gcc` ทำทุกอย่างในคำสั่งเดียว:

```bash
gcc -o main main.c helper.c -lm
```

> **คำถาม 5.1:** คำสั่งข้างบนทำอะไรให้เราอัตโนมัติบ้าง? (อ้างอิง Part 2–4)
>
> ```
> ตอบ:
>
>
> ```

---
