# Lab 2: System Calls บน Linux
---

## วัตถุประสงค์

1. เข้าใจว่า `printf()` / `scanf()` เรียก system call อะไรอยู่เบื้องหลัง
2. ใช้ `strace` ติดตาม system call ได้
3. เข้าใจ file descriptor และ file I/O syscalls เบื้องต้น
4. เข้าใจ process syscalls: `fork()`, `exec()`, `wait()`

---

## สิ่งที่ต้องเตรียม

```bash
sudo apt update && sudo apt install gcc strace -y
mkdir -p ~/syscall_lab && cd ~/syscall_lab
```

---

## Part 1 — strace กับ Library Function

### 1.1 Hello World + strace

สร้างไฟล์ `hello.c`:

```c
#include <stdio.h>
int main(void) {
    printf("Hello, World!\n");
    return 0;
}
```

คอมไพล์และรัน:

```bash
gcc -Wall -o hello hello.c
./hello
```

ใช้ `strace` ดู syscall ที่เกิดขึ้นจริง:

```bash
strace -e trace=write ./hello
```

> **คำถาม 1.1:** `write(1, "Hello, World!\n", 14)` — เลข `1` คืออะไร?
>
> ```
> ตอบ:
>
>
> ```

### 1.2 scanf + strace

สร้างไฟล์ `greet.c`:

```c
#include <stdio.h>
int main(void) {
    char name[64];
    printf("Name? ");
    scanf("%63s", name);
    printf("Hello, %s!\n", name);
    return 0;
}
```

```bash
gcc -Wall -o greet greet.c
strace -e trace=read,write ./greet
```

> **คำถาม 1.2:** `scanf()` เรียก syscall อะไร? fd เป็นเลขอะไร?
>
> ```
> ตอบ:
>
>
> ```

### 1.3 strace กับคำสั่ง Linux ที่มีอยู่แล้ว

ไม่ต้องเขียนโค้ด — ลองใช้ strace กับคำสั่งที่มีอยู่:

```bash
strace -e trace=write echo "test"
```

```bash
strace -e trace=read,write cat /etc/hostname
```

```bash
echo "10 3" > nums.txt
strace -e trace=openat,read,write,close cat nums.txt
```

> **คำถาม 1.3:** `cat` เปิดไฟล์ด้วย syscall อะไร? อ่านด้วยอะไร? พิมพ์ออกจอด้วยอะไร?
>
> ```
> ตอบ:
>
>
> ```

### สรุป Part 1

```
  User Program
       │
       ▼
  ┌──────────────────┐
  │  Library (libc)  │   ← printf(), scanf(), fopen() ...
  │    User Space    │
  └──────────────────┘
       │  system call
       ▼
  ┌──────────────────┐
  │     Kernel       │   ← write(), read(), open() ...
  │   Kernel Space   │
  └──────────────────┘
```

| Library Function | System Call ที่เรียกข้างใน |
|:-----------------|:--------------------------|
| `printf()`       | `write()`                 |
| `scanf()`        | `read()`                  |
| `fopen()`        | `openat()`                |
| `fclose()`       | `close()`                 |

---

## Part 2 — File I/O ด้วย System Call

### 2.1 เปรียบเทียบ: Library vs Syscall

สร้างไฟล์ `write_test.c`:

```c
#include <stdio.h>
#include <fcntl.h>
#include <unistd.h>
int main(void) {
    /* Library function */
    FILE *fp = fopen("lib.txt", "w");
    fprintf(fp, "from library\n");
    fclose(fp);

    /* System call */
    int fd = open("sys.txt", O_WRONLY|O_CREAT|O_TRUNC, 0644);
    write(fd, "from syscall\n", 13);
    close(fd);
    return 0;
}
```

```bash
gcc -Wall -o write_test write_test.c
strace -e trace=openat,write,close ./write_test
cat lib.txt
cat sys.txt
```

> **คำถาม 2.1:** ทั้ง `fopen` และ `open` สุดท้ายเรียก syscall ตัวเดียวกันหรือไม่? ตัวไหน?
>
> ```
> ตอบ:
>
>
> ```

### 2.2 แบบฝึกหัด — เติม fd ให้ถูก

**โจทย์:** เติม `_____` (2 จุด) ให้โปรแกรม copy ไฟล์ทำงานได้

```c
/* mycp.c */
#include <fcntl.h>
#include <unistd.h>
#include <stdio.h>
int main(int argc, char *argv[]) {
    if (argc != 3) { printf("Usage: mycp src dst\n"); return 1; }
    int src = open(argv[1], O_RDONLY);
    int dst = open(argv[2], O_WRONLY|O_CREAT|O_TRUNC, 0644);
    char buf[4096];
    ssize_t n;
    while ((n = read(_____, buf, 4096)) > 0)
        write(_____, buf, n);
    close(src);
    close(dst);
    return 0;
}
```

> **Hint:** `read()` อ่านจาก fd ไหน? `write()` เขียนไป fd ไหน?

ทดสอบ:

```bash
gcc -Wall -o mycp mycp.c
echo "Hello OS Lab" > sample.txt
./mycp sample.txt copy.txt
diff sample.txt copy.txt && echo "PASS"
```

> **คำถาม 2.2:** ใช้ strace ดู — `read()` และ `write()` ถูกเรียกอย่างละกี่ครั้ง? ทำไม?
>
> ```
> strace -e trace=openat,read,write,close ./mycp sample.txt copy2.txt
> ```
>
> ```
> ตอบ:
>
>
> ```

---

## Part 3 — Process System Calls

### 3.1 ทฤษฎี: fork / exec / wait

```
  Parent (PID=100)
      │
      ├── fork() ────► Child (PID=101)    ← copy ของ parent
      │                    │
      │                    └── exec("ls")  ← แทนที่ด้วยโปรแกรมใหม่
      │
      └── wait() ← รอ child จบ
```

| System Call   | หน้าที่ |
|:--------------|:--------|
| `fork()`      | สร้าง process ลูก (สำเนาของแม่) |
| `exec()`      | แทนที่โปรแกรมปัจจุบันด้วยโปรแกรมใหม่ |
| `wait()`      | รอ process ลูกจบ |

### 3.2 ตัวอย่าง: fork + exec + wait

สร้างไฟล์ `run_cmd.c`:

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/wait.h>
int main(void) {
    pid_t pid = fork();
    if (pid == 0) {
        printf("[Child  PID=%d] running ls...\n", getpid());
        execlp("ls", "ls", "-l", NULL);
        perror("exec failed");
        exit(1);
    }
    wait(NULL);
    printf("[Parent PID=%d] child done\n", getpid());
    return 0;
}
```

```bash
gcc -Wall -o run_cmd run_cmd.c
./run_cmd
```

**ลองเปลี่ยนคำสั่ง** — แก้ `execlp("ls", "ls", "-l", NULL)` เป็น:

```c
execlp("date", "date", NULL);           /* แสดงวันที่ */
execlp("whoami", "whoami", NULL);       /* แสดง username */
execlp("xyznotfound", "xyznotfound", NULL);  /* คำสั่งผิด */
```

> **คำถาม 3.1:** ถ้าใส่คำสั่งผิด (xyznotfound) เกิดอะไรขึ้น? ทำไม `perror` ถึงทำงาน?
>
> ```
> ตอบ:
>
>
> ```

> **คำถาม 3.2:** ถ้า `exec` สำเร็จ โค้ดบรรทัดหลัง exec จะถูกรันไหม? ทำไม?
>
> ```
> ตอบ:
>
>
> ```

### 3.3 strace ดู process syscalls

```bash
strace -f -e trace=clone,execve,wait4 ./run_cmd
```

> **คำถาม 3.3:** `fork()` จริงๆ แล้วเรียก syscall ชื่ออะไร? (`-f` หมายถึงอะไร?)
>
> ```
> ตอบ:
>
>
> ```

---

## Part 4 — สรุป

### สรุปสิ่งที่เรียน

```
printf("hello")  →  write(1, "hello", 5)    ← Kernel ส่งไป terminal
scanf("%d", &x)  →  read(0, buf, ...)       ← Kernel อ่านจาก keyboard
fopen("f.txt")   →  openat(...)             ← Kernel เปิดไฟล์
fork()           →  clone(...)              ← Kernel สร้าง process ใหม่
```

### คำถามท้ายแลป

> **คำถาม 4.1:** `printf()` เรียก syscall อะไร? ทำไมหลาย `printf` อาจกลายเป็น `write` แค่ครั้งเดียว?
>
> ```
> ตอบ:
>
>
> ```

> **คำถาม 4.2:** `open()` คืนค่าอะไร? มันคืออะไรในมุมมองของ kernel?
>
> ```
> ตอบ:
>
>
> ```

> **คำถาม 4.3:** `fork()` คืน 0 หมายความว่าอะไร? คืนค่ามากกว่า 0 ล่ะ?
>
> ```
> ตอบ:
>
>
> ```

> **คำถาม 4.4:** ทำไม `exec` ถึงไม่ return ถ้าสำเร็จ?
>
> ```
> ตอบ:
>
>
> ```

---

### อ้างอิงเพิ่มเติม

- `man 2 open` / `man 2 read` / `man 2 write` — คู่มือ syscall
- `man 2 fork` / `man 2 execve` / `man 2 wait`
- `man 1 strace` — เครื่องมือดู syscall

---
