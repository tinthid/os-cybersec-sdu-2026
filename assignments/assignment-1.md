# Assignment 1: ทบทวน Lab 1–3 [Due Date 11/04/2026]
---

## ข้อ 1 — Boot & Daemon

**1.1)** รันคำสั่งต่อไปนี้:

```bash
dmesg | grep -i "memory" | head -5
```

วาง output แล้วอธิบายสั้นๆ ว่าข้อมูลที่แสดงบอกอะไรเกี่ยวกับขั้นตอน boot ของ kernel

> ```
> ตอบ:
>
>
> ```

**1.2)** รันคำสั่งนี้:

```bash
systemctl list-units --type=service --state=running --no-pager | grep -E "ssh|cron|network|log"
```

จาก output — เลือก service มา **2 ตัว** แล้วอธิบายว่า:
- แต่ละตัวทำหน้าที่อะไร?
- ถ้า service นั้นหยุดทำงาน จะกระทบระบบอย่างไร?

> ```
> ตอบ:
>
>
> ```

**1.3)** Bootstrap program (เช่น GRUB) กับ Kernel ต่างกันอย่างไร? อธิบายโดยเปรียบเทียบหน้าที่ของทั้งสอง

> ```
> ตอบ:
>
>
> ```

---

## ข้อ 2 — System Call & strace

**2.1)** สร้างไฟล์ `hw_count.c`:

```c
#include <stdio.h>

int main(void)
{
    for (int i = 1; i <= 3; i++) {
        fprintf(stderr, "error %d\n", i);
        printf("line %d\n", i);
    }
    return 0;
}
```

คอมไพล์แล้วรัน:

```bash
gcc -Wall -o hw_count hw_count.c
strace -e trace=write ./hw_count 2>&1 | tail -20
```

วาง output แล้วตอบ:
- `write(1, ...)` กับ `write(2, ...)` ต่างกันอย่างไร? เลข 1 และ 2 หมายถึงอะไร?
- `printf` กับ `fprintf(stderr, ...)` ใช้ file descriptor ต่างกันอย่างไร?

> ```
> ตอบ:
>
>
> ```

**2.2)** สร้างไฟล์ `hw_copy.c` ที่ใช้ **system call โดยตรง** (ห้ามใช้ `printf` / `fopen` / `fread` / `fwrite`) ทำสิ่งต่อไปนี้:
1. เปิดไฟล์ `/etc/hostname` เพื่ออ่าน
2. อ่านเนื้อหาทั้งหมด
3. สร้างไฟล์ใหม่ชื่อ `hw_hostname_backup.txt` แล้วเขียนเนื้อหาที่อ่านมาลงไป
4. ปิดไฟล์ทั้งสอง

**Hint:** ใช้ `open()`, `read()`, `write()`, `close()` — header ที่ต้องใช้: `<fcntl.h>`, `<unistd.h>`

```bash
gcc -Wall -o hw_copy hw_copy.c
./hw_copy
cat hw_hostname_backup.txt
```

วาง source code ที่เขียน และผลลัพธ์ของ `cat hw_hostname_backup.txt`:

> ```
> ตอบ:
>
>
> ```

**2.3)** อธิบายว่าทำไมโปรแกรมภาษา C ที่ใช้ `printf()` ถึงทำงานได้ทั้งบน Linux และ Windows แต่โปรแกรมที่ใช้ `write()` โดยตรงจะรันได้เฉพาะบน Linux?

> ```
> ตอบ:
>
>
> ```

---

## ข้อ 3 — Process: fork & exec

**3.1)** สร้างไฟล์ `hw_chain.c`:

```c
#include <stdio.h>
#include <unistd.h>
#include <sys/wait.h>

int main(void)
{
    int x = 100;
    pid_t pid = fork();

    if (pid == 0) {
        /* Child */
        x += 50;
        printf("[Child]  PID=%d  x=%d\n", getpid(), x);
    } else {
        /* Parent */
        wait(NULL);
        printf("[Parent] PID=%d  x=%d\n", getpid(), x);
    }

    return 0;
}
```

คอมไพล์แล้วรัน:

```bash
gcc -Wall -o hw_chain hw_chain.c
./hw_chain
```

วาง output แล้วตอบ:
- ค่า `x` ของ Parent กับ Child เท่ากันหรือไม่? ทำไม?
- `wait(NULL)` ทำให้ใครพิมพ์ออกมาก่อน? ถ้าเอาออกจะเป็นอย่างไร?

> ```
> ตอบ:
>
>
> ```

**3.2)** สร้างไฟล์ `hw_exec.c`:

```c
#include <stdio.h>
#include <unistd.h>
#include <sys/wait.h>

int main(void)
{
    pid_t pid = fork();

    if (pid == 0) {
        printf("[Child] Before exec, PID=%d\n", getpid());
        execlp("date", "date", "+%Y-%m-%d %H:%M:%S", NULL);
        printf("[Child] This line should NOT appear\n");
    } else {
        wait(NULL);
        printf("[Parent] Child finished\n");
    }

    return 0;
}
```

คอมไพล์แล้วรัน:

```bash
gcc -Wall -o hw_exec hw_exec.c
./hw_exec
```

วาง output แล้วตอบ: ทำไมบรรทัด `"This line should NOT appear"` ถึงไม่ถูกพิมพ์ออกมา? `execlp()` ทำอะไรกับ process?

> ```
> ตอบ:
>
>
> ```

**3.3)** ถ้า parent process ไม่เรียก `wait()` และจบการทำงานไปก่อน child — จะเกิดอะไรขึ้นกับ child process? คำว่า **orphan process** และ **zombie process** ต่างกันอย่างไร?

> ```
> ตอบ:
>
>
> ```

---

## ข้อ 4 — Compile & Link

**4.1)** สร้าง 2 ไฟล์ต่อไปนี้:

`hw_util.c`:

```c
#include <stdio.h>

void show_info(const char *name, int id)
{
    printf("Name: %s, ID: %d\n", name, id);
}
```

`hw_main.c`:

```c
void show_info(const char *name, int id);  /* ประกาศ prototype */

int main(void)
{
    show_info("Student", 12345);
    return 0;
}
```

ลอง compile และ link แบบแยกขั้นตอน:

```bash
gcc -c hw_main.c -o hw_main.o
gcc -c hw_util.c -o hw_util.o
gcc -o hw_app hw_main.o hw_util.o
./hw_app
```

วาง output แล้วตอบ:
- ทำไมต้อง compile แยกเป็น `.o` ก่อนแล้วค่อย link? ข้อดีคืออะไรเมื่อเทียบกับ compile ทีเดียว?
- ถ้าเราลืมใส่ `hw_util.o` ตอน link (เช่น `gcc -o hw_app hw_main.o`) จะเกิด error อะไร?

> ```
> ตอบ:
>
>
> ```

**4.2)** รันคำสั่งนี้กับไฟล์ `.o` ที่ได้:

```bash
nm hw_main.o
nm hw_util.o
```

วาง output แล้วอธิบาย:
- ตัวอักษร `T` และ `U` ที่อยู่ข้างหน้าชื่อ symbol หมายความว่าอะไร?
- symbol `show_info` ปรากฏแบบ `U` ในไฟล์ไหน? แบบ `T` ในไฟล์ไหน? เพราะอะไร?

> ```
> ตอบ:
>
>
> ```

**4.3)** รันคำสั่ง:

```bash
ldd hw_app
file hw_app
```

วาง output แล้วตอบ: โปรแกรมนี้เป็น statically linked หรือ dynamically linked? สังเกตจากตรงไหน?

> ```
> ตอบ:
>
>
> ```

---
