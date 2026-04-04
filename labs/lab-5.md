# Lab 5: Processes — สำรวจ สร้าง สังเกต
---

## วัตถุประสงค์

1. เข้าใจแนวคิด Process ว่าคืออะไร มีส่วนประกอบอะไรบ้าง
2. ใช้คำสั่ง Linux (`ps`, `pstree`, `top`, `strace`) สำรวจ process จริงในระบบได้
3. เข้าใจ process states และสังเกตได้จาก `/proc`
4. เข้าใจ `fork()`, `exec()`, `wait()` ผ่าน snippet เล็ก ๆ และ `strace`
5. เข้าใจปัญหา zombie / orphan process
6. เข้าใจ IPC (pipe) จากคำสั่ง shell ที่ใช้กันทุกวัน

---

## สิ่งที่ต้องเตรียม

- ระบบ Linux (Ubuntu/Debian หรือ WSL)
- ติดตั้ง tools:

```bash
sudo apt update
sudo apt install gcc build-essential strace htop -y
```

สร้างโฟลเดอร์สำหรับแลป:

```bash
mkdir -p ~/lab_process && cd ~/lab_process
```

---

## ความรู้พื้นฐาน — Process คืออะไร?

ลองนึกภาพแบบนี้ — สมมุติว่าเรามี **สูตรทำเค้ก** เขียนอยู่ในหนังสือ ตัวสูตรมันแค่นอนอยู่ในหนังสือเฉย ๆ ไม่ได้ทำอะไร นั่นคือ **program** แต่พอเราหยิบสูตรมา เปิดเตาอบ ตักแป้ง ผสมไข่ ลงมือทำจริง ๆ กระบวนการทำเค้กทั้งหมดนั่นคือ **process**

**Process = program ที่กำลังทำงาน** (a program in execution)

### ส่วนประกอบของ Process

Process ไม่ใช่แค่ code อย่างเดียว มันมีหลายส่วน:

```
┌─────────────────────┐  ← max address
│       Stack          │  ↓ โตลงล่าง
│  (local variables,   │
│   function calls)    │
│        ↓             │
│                      │
│     (พื้นที่ว่าง)     │
│                      │
│        ↑             │
│       Heap           │  ↑ โตขึ้นบน
│  (malloc, new)       │
├──────────────────────┤
│       Data           │  global variables, static variables
├──────────────────────┤
│       Text           │  program code (read-only)
└──────────────────────┘  ← address 0
```

แต่ละส่วนทำหน้าที่ต่างกัน:

| ส่วน | เก็บอะไร | ตัวอย่าง |
|---|---|---|
| **Text** | machine code ของโปรแกรม (read-only) | คำสั่งที่ CPU จะ execute |
| **Data** | global variables | `int count = 0;` ที่ประกาศนอก function |
| **Heap** | memory ที่จองแบบ dynamic | `malloc(100)` หรือ `new` |
| **Stack** | local variables, function calls | `int x = 5;` ที่ประกาศใน function |

นอกจากนี้ OS ยังเก็บข้อมูลเพิ่มเติมสำหรับ process แต่ละตัวใน **PCB (Process Control Block)**:

- **Program counter** — ชี้ว่ากำลัง execute คำสั่งไหน
- **Process state** — ตอนนี้อยู่ใน state อะไร
- **CPU registers** — ค่าใน register ทั้งหมด
- **PID** — หมายเลขประจำตัว process

> **จุดสำคัญ:** program 1 ตัวสามารถเป็นหลาย process ได้ เช่น ถ้าเปิด terminal 3 หน้าต่าง ก็จะมี 3 process ของ bash ทำงานพร้อมกัน แต่ใช้ program `/bin/bash` ตัวเดียวกัน

### Process States — 5 สถานะของ Process

Process ไม่ได้อยู่ในสถานะเดียวตลอด มันจะเปลี่ยนไปเรื่อย ๆ:

```
          admitted              scheduler dispatch
  New ──────────→ Ready ─────────────────→ Running
                    ↑                        │  │
                    │    interrupt            │  │
                    ├────────────────────────←┘  │
                    │                            │ exit
                    │  I/O or event              ↓
                    │  completion          Terminated
                    │       ↑
                    │       │
                  Waiting ←─┘
                     I/O or event wait
```

| State | ความหมาย | เปรียบเทียบ |
|---|---|---|
| **New** | process เพิ่งถูกสร้าง | เพิ่งลงทะเบียนเรียน |
| **Ready** | พร้อมทำงาน แค่รอ CPU ว่าง | สั่งอาหารเสร็จ รอพนักงานเสิร์ฟ |
| **Running** | กำลังใช้ CPU อยู่จริง ๆ | กำลังกินอาหาร |
| **Waiting** | รอ event บางอย่าง (เช่น I/O) | อาหารยังทำไม่เสร็จ |
| **Terminated** | ทำงานเสร็จแล้ว | กินเสร็จ จ่ายตังค์แล้ว |

> **จุดที่นักศึกษาสับสนบ่อย:** Ready กับ Waiting ต่างกันยังไง?
> - **Ready** = พร้อมแล้ว แค่รอ CPU (พนักงานเสิร์ฟ) ว่าง
> - **Waiting** = ยังไม่พร้อม ต้องรอ I/O เสร็จก่อน (อาหารยังทำไม่เสร็จ)
>
> **สังเกต:** ไม่มีลูกศรจาก Waiting ไป Running ตรง ๆ — ต้องผ่าน Ready ก่อนเสมอ!

---

## Part 1 — สำรวจ Process ในระบบ

เราจะเริ่มจากการดู process ที่มีอยู่แล้วในระบบก่อน โดย **ไม่ต้องเขียนโค้ดเลย**

### 1.1 ดู process ที่กำลังทำงาน

คำสั่ง `ps` (process status) ใช้แสดง process ที่ทำงานอยู่:

```bash
ps aux | head -20
```

ผลลัพธ์จะมีหลาย column:

```
USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root         1  0.0  0.1  22556  9988 ?        Ss   10:00   0:01 /sbin/init
root         2  0.0  0.0      0     0 ?        S    10:00   0:00 [kthreadd]
...
```

| Column | ความหมาย |
|---|---|
| `USER` | เจ้าของ process |
| `PID` | หมายเลขประจำตัว process |
| `%CPU` | ใช้ CPU กี่ % |
| `%MEM` | ใช้ memory กี่ % |
| `STAT` | สถานะ (S=sleeping, R=running, Z=zombie) |
| `COMMAND` | คำสั่งที่รัน |

ดู process ของตัวเองเท่านั้น:

```bash
ps -u $USER
```

### 1.2 ดู process tree

ใน Linux process ทุกตัวมี **parent** ยกเว้น PID 1 ที่เป็นรากของต้นไม้:

```bash
pstree -p | head -30
```

ผลลัพธ์จะแสดง tree structure แบบนี้:

```
systemd(1)─┬─login(500)───bash(600)───pstree(700)
            ├─sshd(200)───sshd(300)───bash(400)
            └─cron(100)
```

> **สังเกต:** ทุก process มี parent — เหมือนต้นไม้ครอบครัว process ตัวบนสุดคือ `systemd` (PID 1) ซึ่งเป็น process ตัวแรกที่ OS สร้างตอน boot

> **คำถาม 1.1:** process ตัวแรกสุดของระบบ (PID 1) ชื่ออะไร? ทำหน้าที่อะไร?
>
> ```
> ตอบ:
>
>
> ```

### 1.3 ดู PID ของ shell ปัจจุบัน

ตัวแปรพิเศษ `$$` ใน bash เก็บ PID ของ shell ปัจจุบัน:

```bash
echo "My shell PID = $$"
```

ทุกครั้งที่เราพิมพ์คำสั่ง เช่น `ls` → bash จะสร้าง child process ขึ้นมาเพื่อรันคำสั่งนั้น ลองพิสูจน์:

```bash
echo "Shell PID = $$"
bash -c 'echo "Command PID = $$, Parent PID = $PPID"'
```

### 1.4 สำรวจ /proc filesystem

Linux มี pseudo-filesystem ชื่อ `/proc` ที่ kernel ใช้เปิดเผยข้อมูลของ process ให้อ่านได้ ทุก process จะมีโฟลเดอร์ `/proc/<PID>/` ของตัวเอง

ดูข้อมูลของ shell ปัจจุบัน:

```bash
ls /proc/$$/
```

> **สังเกต:** มีไฟล์เยอะมาก แต่ละไฟล์เก็บข้อมูลคนละอย่างเกี่ยวกับ process นี้

ดูข้อมูลสถานะ:

```bash
cat /proc/$$/status | head -20
```

ผลลัพธ์จะแสดงข้อมูลคล้าย ๆ PCB:

```
Name:   bash
State:  S (sleeping)
Tgid:   1234
Pid:    1234
PPid:   1000
...
VmSize: 12000 kB
VmRSS:  5000 kB
Threads: 1
```

| ฟิลด์ | ความหมาย | เทียบกับ PCB |
|---|---|---|
| `Name` | ชื่อโปรแกรม | program name |
| `State` | สถานะปัจจุบัน | process state |
| `Pid` | หมายเลข process | PID |
| `PPid` | PID ของ parent | parent pointer |
| `VmSize` | memory ทั้งหมดที่ใช้ | memory management info |
| `Threads` | จำนวน thread | thread count |

> **คำถาม 1.2:** `/proc/$$/status` แสดงข้อมูลอะไรบ้าง? ฟิลด์ `State` ตรงกับ process state ไหนในทฤษฎี?
>
> ```
> ตอบ:
>
>
> ```

ดู memory map ของ process:

```bash
cat /proc/$$/maps | head -10
```

> **สังเกต:** จะเห็น address ของ text, data, heap, stack ตรงกับรูป memory layout ที่เรียนในทฤษฎี

> **คำถาม 1.3:** ในผลลัพธ์ของ `/proc/$$/maps` มี section ที่ระบุว่า `[heap]` และ `[stack]` หรือไม่? address ของ stack สูงกว่า heap หรือไม่? ตรงกับทฤษฎีหรือไม่?
>
> ```
> ตอบ:
>
>
> ```

### 1.5 นับจำนวน process ในระบบ

```bash
ps aux | wc -l
```

> **คำถาม 1.4:** ในระบบของนักศึกษามี process ทั้งหมดกี่ตัว? คิดว่าทำไมถึงมีเยอะขนาดนี้ ทั้ง ๆ ที่เราแค่เปิด terminal ตัวเดียว?
>
> ```
> ตอบ:
>
>
> ```

---

## Part 2 — สังเกต Process States จริงด้วย Tools

ในทฤษฎีเราเรียนว่า process มี 5 states — ตอนนี้จะมาสังเกตจริง ๆ ในระบบ

### 2.1 ดู state ของ process ด้วย `ps`

ใน `ps aux` คอลัมน์ `STAT` แสดงสถานะจริงของ process:

| STAT code | ความหมาย | ตรงกับทฤษฎี |
|---|---|---|
| `R` | Running / Runnable | **Running** หรือ **Ready** |
| `S` | Sleeping (interruptible) | **Waiting** (รอ I/O หรือ event) |
| `D` | Disk sleep (uninterruptible) | **Waiting** (รอ I/O จริง ๆ เช่น อ่าน disk) |
| `T` | Stopped | ถูก pause ด้วย signal |
| `Z` | Zombie | **Terminated** แต่ parent ยังไม่เก็บ status |

> **สังเกต:** Linux ไม่มี state `R` แยก Running กับ Ready ออกจากกัน — ใช้ `R` ทั้งสองอย่าง เพราะ kernel ดูแลการ schedule เอง

ลองดู process ที่มี state ต่าง ๆ:

```bash
ps aux | awk '{print $8}' | sort | uniq -c | sort -rn
```

> **คำถาม 2.1:** state ไหนมีจำนวน process มากที่สุด? ทำไม? 
>
> ```
> ตอบ:
>
>
> ```

### 2.2 สังเกต process state เปลี่ยนแปลง

เปิด terminal 2 หน้าต่าง

**Terminal 1:** รัน `sleep` (process ที่จะอยู่ใน state S ตลอด):

```bash
sleep 60
```

**Terminal 2:** ค้นหา sleep process แล้วดู state:

```bash
ps aux | grep "sleep 60" | grep -v grep
```

> **สังเกต:** `sleep` อยู่ใน state `S` (sleeping) ตลอด — เพราะมันรอ timer ไม่ได้ใช้ CPU เลย

ลองหยุด process ด้วย `Ctrl+Z` ใน terminal 1 แล้วดู state อีกครั้ง:

```bash
# กลับไป terminal 1 กด Ctrl+Z
# แล้วใน terminal 2:
ps aux | grep "sleep 60" | grep -v grep
```

> **คำถาม 2.2:** state เปลี่ยนจากอะไรเป็นอะไรเมื่อกด `Ctrl+Z`? state `T` ตรงกับทฤษฎีอะไร?
>
> ```
> ตอบ:
>
>
> ```

หลังจากกด `Ctrl+Z` process จะถูก suspend และเราจะได้ prompt กลับมา — ลองใช้ `fg` เพื่อดึง process กลับมาทำงานต่อใน foreground:

```bash
# ใน terminal 1 (หลังกด Ctrl+Z แล้ว):
fg
```

### 2.3 ใช้ `top` ดู process แบบ real-time

```bash
top
```

กดปุ่มต่อไปนี้ขณะ `top` ทำงาน:

| ปุ่ม | ทำอะไร |
|---|---|
| `P` | เรียงตาม %CPU |
| `M` | เรียงตาม %MEM |
| `k` | kill process (พิมพ์ PID) |
| `q` | ออก |

> **คำถาม 2.3:** ตอนที่เปิด `top` process ไหนใช้ CPU มากที่สุด? ทำไม?
>
> ```
> ตอบ:
>
>
> ```

### 2.4 สร้าง process ที่ใช้ CPU เยอะแล้วสังเกต

ลองสร้าง load ให้ CPU ดู:

```bash
# Terminal 1: สร้าง CPU load
yes > /dev/null &
echo "PID = $!"
```

```bash
# Terminal 2: ดูด้วย top
top -p <PID ที่ได้>
```

> **สังเกต:** process `yes` จะใช้ CPU เกือบ 100% และอยู่ใน state `R` (running)

```bash
# อย่าลืม kill ทิ้ง!
kill <PID>
```

> **คำถาม 2.4:** process `yes` อยู่ใน state อะไร? ทำไมถึงใช้ CPU เกือบ 100%?
>
> ```
> ตอบ:
>
>
> ```

---

## Part 3 — fork, exec, wait — เข้าใจผ่าน strace

### Shell ทำงานยังไง?

ทุกครั้งที่เราพิมพ์คำสั่งใน bash เช่น `ls -la` สิ่งที่เกิดขึ้นเบื้องหลังคือ:

```
  1. bash เรียก fork()     → ได้ child process
  2. child เรียก exec("ls") → child กลายเป็น ls
  3. ls ทำงาน แสดงไฟล์      → ls เรียก exit()
  4. bash เรียก wait()      → bash รู้ว่า child เสร็จแล้ว
  5. bash แสดง prompt $     → พร้อมรับคำสั่งต่อไป
```

```
  bash (parent)
    │
    ├── fork() ──→ Child (copy of bash)
    │                │
    │                ├── exec("ls") ──→ ls ทำงาน ──→ exit()
    │
    └── wait() ──→ รอ child เสร็จ ──→ แสดง prompt $ ใหม่
```

pattern `fork → exec → wait` นี้สำคัญมาก — เป็นวิธีที่ shell ทุกตัวใน Unix/Linux ใช้รันคำสั่ง

### 3.1 พิสูจน์ด้วย strace — ดู system call จริง

`strace` คือเครื่องมือที่แสดง **ทุก system call** ที่ process เรียก เราจะใช้มันพิสูจน์ว่า bash ใช้ fork → exec → wait จริง:

```bash
strace -f -e trace=clone,execve,wait4 bash -c "ls /tmp; true"
```

อธิบาย option:
- `-f` = ติดตาม child process ด้วย
- `-e trace=clone,execve,wait4` = แสดงเฉพาะ system call ที่เราสนใจ
- `bash -c "ls /tmp; true"` = เปิด bash ใหม่ แล้วรัน `ls /tmp` ตามด้วย `true`

> **ทำไมต้องมี `; true`?** ถ้ามีคำสั่งเดียว bash จะ optimize โดย exec ทับตัวเองเลยโดยไม่ fork child ใหม่ การเพิ่ม `; true` บังคับให้ bash ต้อง fork จริงๆ เพราะยังมีคำสั่งถัดไปต้องทำ

> **สังเกต:** ใน output จะเห็น:
> 1. `clone(...)` — bash สร้าง child (clone คือ fork เวอร์ชัน Linux)
> 2. `execve("/bin/ls", ...)` — child โหลดโปรแกรม ls
> 3. `wait4(...)` — bash รอ child เสร็จ

> **คำถาม 3.1:** จาก output ของ strace เห็น system call อะไรบ้าง? ตรงกับ pattern `fork → exec → wait` หรือไม่?
>
> ```
> ตอบ:
>
>
> ```

### 3.2 strace กับ pipe `|`

ลองดูว่าเกิดอะไรเมื่อใช้ pipe:

```bash
strace -f -e trace=clone,execve,pipe2,dup2,wait4 bash -c "ls | grep .c" 2>&1 | head -30
```

> **สังเกต:** จะเห็น:
> 1. `pipe2(...)` — bash สร้าง pipe ก่อน (Linux ใช้ `pipe2` แทน `pipe` ธรรมดา)
> 2. `clone(...)` — สร้าง child ตัวแรก (ls)
> 3. `clone(...)` — สร้าง child ตัวที่สอง (grep)
> 4. `dup2(...)` — redirect stdout ของ ls เข้า pipe, redirect stdin ของ grep ออกจาก pipe
> 5. `execve(...)` — ทั้งคู่โหลดโปรแกรม

> **คำถาม 3.2:** เมื่อพิมพ์ `ls | grep .c` bash สร้าง process กี่ตัว? ใช้ pipe กี่ตัว?
>
> ```
> ตอบ:
>
>
> ```

### 3.3 เข้าใจ fork() — process ลูกได้ copy ของ memory

`fork()` สร้าง child process ที่เป็น **copy** ของ parent — ทุกอย่างเหมือนกันหมด ยกเว้น:
- `fork()` return ค่าต่างกัน: **> 0** ให้ parent (เป็น PID ของ child), **0** ให้ child

```
          ก่อน fork()                       หลัง fork()
     ┌──────────────┐              ┌──────────────┐   ┌──────────────┐
     │   Process    │              │   Parent     │   │    Child     │
     │              │   fork()     │              │   │  (copy ของ   │
     │  x = 10     │ ──────────→  │  x = 10     │   │   parent)    │
     │  code...    │              │  code...    │   │  x = 10     │
     └──────────────┘              │  fork()=1234 │   │  fork()=0    │
                                   └──────────────┘   └──────────────┘
                                   return child PID    return 0
```

สร้างไฟล์ `fork_demo.c` แล้ว compile และรัน:

```c
#include <stdio.h>
#include <unistd.h>
#include <sys/wait.h>

int x = 10;  /* global variable */

int main() {
    printf("Before fork: x = %d, PID = %d\n", x, getpid());

    pid_t pid = fork();  /* ← ณ จุดนี้มี 2 process แล้ว! */

    if (pid == 0) {
        x = 99;
        printf("Child:  x = %d, PID = %d\n", x, getpid());
    } else {
        wait(NULL);
        printf("Parent: x = %d, PID = %d\n", x, getpid());
    }
    return 0;
}
```

```bash
gcc -Wall fork_demo.c -o fork_demo
./fork_demo
```

ตัวอย่างผลลัพธ์:

```
Before fork: x = 10, PID = 5000
Child:  x = 99, PID = 5001
Parent: x = 10, PID = 5000
```

> **สังเกต:** child เปลี่ยน x เป็น 99 แต่ parent ยังเห็น x = 10 เพราะ child ได้ **copy** ของ memory ไม่ใช่ memory ตัวเดียวกัน

ลองใช้ `strace` ดู system call ที่เกิดขึ้น:

```bash
strace -f -e trace=clone,write,wait4 ./fork_demo 2>&1
```

> **คำถาม 3.3:** จาก output ของ `strace` จะเห็น `clone(...)` ไหม? child process มี PID เท่าไหร่?
>
> ```
> ตอบ:
>
>
> ```

> **คำถาม 3.4:** child เปลี่ยน x เป็น 99 แล้ว parent เห็นค่า x เท่าไหร่? สิ่งนี้พิสูจน์อะไรเกี่ยวกับ memory ของ process?
>
> ```
> ตอบ:
>
>
> ```

### 3.4 เข้าใจ exec() — แทนที่โปรแกรมทั้งหมด

`exec()` **แทนที่** memory ของ process ปัจจุบันด้วยโปรแกรมใหม่ทั้งหมด:

```
  ก่อน exec()                 หลัง exec("ls")
┌──────────────┐           ┌──────────────┐
│  Child       │           │  Child       │
│  (copy of    │  exec()   │  (ตอนนี้คือ   │
│   parent)    │ ────────→ │   ls แล้ว)    │
│  code ของ    │           │  code ของ    │
│  parent      │           │  ls          │
└──────────────┘           └──────────────┘
  PID ไม่เปลี่ยน!            PID เดิม!
```

> **จุดสำคัญ:** ถ้า `exec()` สำเร็จ มันจะ **ไม่ return** เลย! เพราะ code เดิมถูกแทนที่หมดแล้ว

ลองพิสูจน์ด้วย `strace` — ดูว่าเมื่อ bash รัน `date` มันใช้ exec อะไร:

```bash
strace -f -e trace=clone,execve bash -c "date" 2>&1 | grep execve
```

> **คำถาม 3.5:** `execve` ถูกเรียกกี่ครั้ง? ครั้งแรกเรียกโปรแกรมอะไร? ครั้งที่สองเรียกโปรแกรมอะไร?
>
> ```
> ตอบ:
>
>
> ```

### 3.5 ดู pstree ขณะ process ทำงาน

ลองดู process tree ขณะที่รันคำสั่ง:

```bash
bash -c "pstree -p $$"
```

> **สังเกต:** จะเห็น chain: ... → bash(PID) → bash(PID2) → pstree(PID3)
> bash ตัวแรกคือ shell ที่เราใช้, bash ตัวที่สอง `bash -c ...` คือ child, pstree คือ child ของ child

> **คำถาม 3.6:** ผลลัพธ์แสดง process tree เป็นอย่างไร? อธิบายว่า parent-child relationship คืออะไร
>
> ```
> ตอบ:
>
>
> ```

### 3.6 fork() หลายครั้ง — คิดเลข ไม่ต้องเขียนโค้ด

ถ้าเรียก `fork()` หลายครั้ง จำนวน process จะเพิ่มขึ้นแบบทวีคูณ:

```
  fork() ครั้งที่ 1:  1 process → 2 processes
  fork() ครั้งที่ 2:  2 processes → 4 processes
  fork() ครั้งที่ 3:  4 processes → 8 processes
```

```
           P (original)
          / \
    fork()1  fork()1
        /      \
       P        C1
      / \      / \
 fork()2 fork()2 fork()2 fork()2
    /      \      /      \
   P       C2    C1      C3
```

> **คำถาม 3.7:** ถ้าเรียก `fork()` 4 ครั้ง จะได้ process ทั้งหมดกี่ตัว? เขียนสูตรทั่วไปสำหรับ n ครั้ง
>
> ```
> ตอบ:
>
>
> ```

---

## Part 4 — Zombie และ Orphan Process

### Zombie Process คืออะไร?

**Zombie** เกิดเมื่อ child terminate แล้ว แต่ parent **ยังไม่ได้เรียก `wait()`** เพื่อเก็บ exit status

```
  เวลา ──────────────────────────────────────→

  Parent:  fork()─────── ทำงานอื่น ─── sleep ─── (ไม่เรียก wait!)
                │
  Child:        └──────→ ทำงาน → exit()
                                    │
                                    ▼
                              Zombie process
                              (ตายแล้ว แต่ยังมี entry ใน process table)
```

Zombie ไม่ใช้ CPU ไม่ใช้ memory แต่ยังกิน **PID slot** ใน process table ถ้ามี zombie เยอะ ๆ อาจทำให้สร้าง process ใหม่ไม่ได้

### 4.1 สร้าง zombie จาก shell

ไม่ต้องเขียน C — ใช้ bash สร้าง zombie ได้เลย:

**Terminal 1:**
```bash
# สร้าง parent ที่ไม่เรียก wait
bash -c '
    bash -c "exit 0" &
    CHILD=$!
    echo "Parent=$$, Child=$CHILD"
    echo "ตรวจสอบใน terminal อื่น: ps aux | grep Z"
    sleep 20
' &
```

**Terminal 2:** ค้นหา zombie:
```bash
ps aux | grep Z | grep -v grep
```

> **สังเกต:** จะเห็น process ที่มี state `Z+` (zombie) มี VSZ และ RSS เป็น 0

> **คำถาม 4.1:** zombie process ใช้ CPU หรือ memory หรือไม่? แล้วทำไมถึงยังเป็นปัญหา?
>
> ```
> ตอบ:
>
>
> ```

> **คำถาม 4.2:** จะป้องกัน zombie ได้อย่างไร? (Hint: parent ควรทำอะไร?)
>
> ```
> ตอบ:
>
>
> ```

### Orphan Process คืออะไร?

**Orphan** เกิดเมื่อ parent terminate **ก่อน** child — child กลายเป็น "เด็กกำพร้า"

```
  เวลา ──────────────────────────────────────→

  Parent:  fork()───→ exit()
                │        │
                │        └── parent ตายแล้ว!
                │
  Child:        └────────────→ ทำงานต่อ... ───→ ใครเป็น parent?
                                                     │
                                                     ▼
                                              systemd (PID 1)
                                              รับเลี้ยงอัตโนมัติ!
```

ใน Linux เมื่อ parent ตาย **systemd (PID 1) จะรับเป็น parent ใหม่** ให้ child อัตโนมัติ

### 4.2 สร้าง orphan จาก shell

```bash
bash -c '
    echo "Parent PID = $$, exiting now..."
    bash -c "
        echo \"Child started, parent = \$PPID\"
        sleep 3
        echo \"Child after 3s, parent = \$(cat /proc/self/status | grep PPid)\"
    " &
'
# รอ 4 วินาทีแล้วจะเห็น output ของ child
sleep 4
```

> **สังเกต:** PPid เปลี่ยนจาก PID ของ parent เดิม ไปเป็น PID 1 (systemd)

> **คำถาม 4.3:** หลังจาก parent ตาย PPid ของ child เปลี่ยนเป็นอะไร? ทำไม?
>
> ```
> ตอบ:
>
>
> ```

> **คำถาม 4.4:** เปรียบเทียบ zombie กับ orphan:
> - อันไหนอันตรายกว่า?
> - OS จัดการอันไหนได้อัตโนมัติ?
>
> ```
> ตอบ:
>
>
> ```

---

## Part 5 — IPC: Pipe — จากคำสั่ง Shell ถึงเบื้องหลัง

### ทำไม Process ต้องสื่อสารกัน?

Process แต่ละตัวมี **memory แยกกัน** (เราพิสูจน์แล้วใน Part 3) จะส่งข้อมูลหากันได้ยังไง?

คำตอบคือ **IPC (Interprocess Communication)** — มีหลายวิธี แต่วิธีที่ใช้บ่อยที่สุดในชีวิตประจำวันคือ **pipe**

### Pipe คืออะไร?

Pipe เหมือน **ท่อน้ำ** ที่มี 2 ปลาย:

```
  Process A                              Process B
  (writer)                               (reader)
     │                                      │
     │   write(fd[1], data)                 │
     └────────────→ ┌────────┐ ────────────→│
                    │  PIPE  │              │
                    │ (buffer│   read(fd[0], buf)
                    │  ใน    │
                    │ kernel)│
                    └────────┘
```

ข้อมูลไหลได้ **ทิศทางเดียว** (unidirectional) เท่านั้น

### 5.1 เราใช้ pipe ทุกวันอยู่แล้ว!

ทุกครั้งที่พิมพ์ `|` ใน terminal เราใช้ pipe:

```bash
ls -la | grep ".c" | wc -l
```

คำสั่งนี้สร้าง 3 process กับ 2 pipe:

```
  ls -la ─── pipe1 ───→ grep ".c" ─── pipe2 ───→ wc -l
```

ลองดูจำนวน process ที่เกิดขึ้น:

```bash
echo "Shell PID = $$"
ps --forest -o pid,ppid,cmd -g $$
```

### 5.2 pipe มี buffer ขนาดจำกัด

pipe ใน kernel มี buffer (ปกติ **65536 bytes** หรือ 64 KB) ลองตรวจสอบ:

```bash
# ดูขนาด pipe buffer ในระบบ
cat /proc/sys/fs/pipe-max-size
```

ลองส่งข้อมูลจำนวนมากผ่าน pipe แล้วดูว่าเกิดอะไร:

```bash
# สร้างข้อมูล 100KB แล้วส่งผ่าน pipe
dd if=/dev/zero bs=1024 count=100 2>/dev/null | wc -c
```

> **คำถาม 5.1:** pipe buffer มีขนาดกี่ bytes? ถ้า writer เขียนเร็วกว่าที่ reader อ่าน จนกว่า buffer จะเต็ม จะเกิดอะไรขึ้น?
>
> ```
> ตอบ:
>
>
> ```

### 5.3 ดูเบื้องหลังของ `|` ด้วย strace

```bash
strace -f -e trace=pipe,clone,dup2,execve,read,write bash -c "echo hello | cat" 2>&1 | head -40
```

> **สังเกต:** จะเห็นลำดับ:
> 1. `pipe([3, 4])` — สร้าง pipe (read end = fd 3, write end = fd 4)
> 2. `clone(...)` — สร้าง child process
> 3. `dup2(4, 1)` — redirect stdout (fd 1) ไปที่ pipe write end (fd 4)
> 4. `dup2(3, 0)` — redirect stdin (fd 0) มาจาก pipe read end (fd 3)
> 5. `execve(...)` — โหลดโปรแกรม

> **คำถาม 5.2:** `dup2(4, 1)` หมายความว่าอะไร? ทำไม `echo` ถึงส่ง output เข้า pipe ได้ ทั้ง ๆ ที่ `echo` ไม่รู้จัก pipe?
>
> ```
> ตอบ:
>
>
> ```

### 5.4 Named pipe (FIFO) — pipe ที่อยู่เป็นไฟล์

ปกติ pipe `|` ใช้ได้เฉพาะระหว่าง parent-child แต่ **named pipe** สร้างเป็นไฟล์ได้ ให้ process ไหนก็ได้เขียน/อ่าน:

```bash
# สร้าง named pipe
mkfifo /tmp/my_pipe
ls -la /tmp/my_pipe
```

> **สังเกต:** ไฟล์มี type `p` (pipe) แทนที่จะเป็น `-` (regular file)

เปิด terminal 2 หน้าต่าง:

**Terminal 1 (reader):**
```bash
cat /tmp/my_pipe
# จะ block รอจนกว่ามีคนเขียน...
```

**Terminal 2 (writer):**
```bash
echo "Hello from terminal 2!" > /tmp/my_pipe
```

> **สังเกต:** พอพิมพ์ echo ใน terminal 2 → ข้อความปรากฏใน terminal 1 ทันที!

```bash
# ลบ named pipe เมื่อเสร็จ
rm /tmp/my_pipe
```

> **คำถาม 5.3:** named pipe ต่างจาก pipe ปกติ (`|`) อย่างไร? มีข้อดีอะไร?
>
> ```
> ตอบ:
>
>
> ```

### 5.5 เปรียบเทียบ IPC วิธีต่าง ๆ

| วิธี IPC | ความเร็ว | ทิศทาง | ข้อจำกัด | ตัวอย่าง |
|---|---|---|---|---|
| **Pipe** (`\|`) | ปานกลาง | ทิศเดียว | parent-child เท่านั้น | `ls \| grep` |
| **Named Pipe** (FIFO) | ปานกลาง | ทิศเดียว | process ไหนก็ได้ | server ↔ client |
| **Shared Memory** | **เร็วมาก** | อ่าน/เขียนทั้งคู่ | ต้อง sync เอง | database shared cache |
| **Signal** | เร็ว | ทิศเดียว | ส่งได้แค่เลข | `kill -9 PID` |

> **คำถาม 5.4:** ถ้าต้องส่งข้อมูลขนาดใหญ่ (เช่น ไฟล์ 1 GB) ระหว่าง 2 process ควรใช้ IPC วิธีไหน? ทำไม?
>
> ```
> ตอบ:
>
>
> ```

---

## สรุปคำสั่งที่ใช้ในแลป

### คำสั่ง Linux

| คำสั่ง | หน้าที่ |
|---|---|
| `ps aux` | แสดง process ทั้งหมดในระบบ |
| `ps -u $USER` | แสดง process ของ user ปัจจุบัน |
| `pstree -p` | แสดง process tree พร้อม PID |
| `top` | ดู process แบบ real-time |
| `strace -f -e trace=...` | ดู system call ของ process |
| `kill PID` | ส่ง signal ให้ process |
| `mkfifo` | สร้าง named pipe |
| `/proc/<PID>/status` | ข้อมูลสถานะของ process |
| `/proc/<PID>/maps` | memory map ของ process |

### System Calls ที่ควรรู้

| System Call | หน้าที่ |
|---|---|
| `fork()` / `clone()` | สร้าง child process (copy จาก parent) |
| `exec()` | โหลดโปรแกรมใหม่ทับ process ปัจจุบัน |
| `wait()` | parent รอ child terminate + เก็บ exit status |
| `exit()` | จบ process พร้อมส่ง exit status |
| `pipe()` | สร้าง pipe สำหรับ IPC |
| `dup2()` | redirect file descriptor |
| `getpid()` / `getppid()` | ดู PID ของตัวเอง / parent |

---

## สรุป

ในแลปนี้เราได้เรียนรู้:

1. **Process** คือ program ที่กำลังทำงาน มีส่วนประกอบ text, data, heap, stack — ดูได้จาก `/proc/<PID>/maps`
2. **Process states** มี 5 สถานะ — ดูสถานะจริงได้จาก `ps aux` (STAT column) และ `/proc/<PID>/status`
3. **Shell** ทำงานด้วย pattern **fork → exec → wait** วนซ้ำ — พิสูจน์ได้ด้วย `strace`
4. `fork()` สร้าง child ที่ได้ **copy** ของ memory — parent กับ child แก้ค่ากันไม่กระทบ
5. `exec()` **แทนที่** memory ทั้งหมดด้วยโปรแกรมใหม่ — ถ้าสำเร็จจะไม่ return
6. **Zombie** = child ตายแล้ว parent ไม่เรียก wait() → กิน PID slot
7. **Orphan** = parent ตายก่อน → systemd (PID 1) รับเลี้ยงอัตโนมัติ
8. **Pipe** คือ IPC แบบท่อทิศทางเดียว — เราใช้มันทุกวันผ่าน `|` ใน shell
9. `strace` เป็นเครื่องมือทรงพลังสำหรับดู system call เบื้องหลังทุกคำสั่ง
