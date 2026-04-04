# Lab 6: Threads & Concurrency — สังเกต เข้าใจ ทดลอง
---

## วัตถุประสงค์

1. เข้าใจแนวคิด Thread ว่าต่างจาก Process อย่างไร
2. สังเกต thread จริงในระบบด้วย `ps`, `htop`, `/proc`
3. เห็นว่า thread **share memory** กัน (ต่างจาก process ที่แยกกัน)
4. เข้าใจ Threading Issues — พฤติกรรมของ **fork()** และ **signal** ใน multithreaded program
5. สังเกต **clone()** system call ที่ Linux ใช้สร้าง thread จริง ๆ

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
mkdir -p ~/lab_thread && cd ~/lab_thread
```

---

## ความรู้พื้นฐาน — Thread คืออะไร?

ใน Lab 5 เราเรียนเรื่อง **Process** — แต่ละ process มี memory แยกกัน ถ้าต้องการสื่อสารต้องใช้ IPC (pipe, shared memory) ซึ่งยุ่งยากพอสมควร

ลองนึกภาพ **Microsoft Word** — ตอนที่เรากำลังพิมพ์อยู่ Word ทำหลายอย่างพร้อมกัน:
- รับ input จากแป้นพิมพ์ แสดงตัวอักษรบนหน้าจอ
- ขีดเส้นแดงใต้คำที่สะกดผิด (spell check)
- Auto-save ไฟล์ทุก ๆ กี่นาที

ถ้าทั้งหมดนี้ทำใน **process เดียว thread เดียว** Word ต้อง:
1. หยุดรับ input → spell check → หยุด spell check → save → หยุด save → กลับมารับ input
2. ผู้ใช้จะรู้สึกว่า Word **ค้าง** ระหว่าง spell check หรือ save

ถ้าใช้ **หลาย thread**:
- Thread 1: รับ input + แสดงผล
- Thread 2: spell check อยู่เบื้องหลัง
- Thread 3: auto-save อยู่เบื้องหลัง

ทั้ง 3 thread ทำงาน **พร้อมกัน** ผู้ใช้ไม่รู้สึกค้างเลย!

### Thread คือ "mini process" ที่อยู่ภายใน process

```
┌───────────────────── Process ─────────────────────┐
│                                                    │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐        │
│  │ Thread 1 │  │ Thread 2 │  │ Thread 3 │        │
│  │          │  │          │  │          │        │
│  │ registers│  │ registers│  │ registers│  ← ของใครของมัน │
│  │ stack    │  │ stack    │  │ stack    │        │
│  │ PC       │  │ PC       │  │ PC       │        │
│  └──────────┘  └──────────┘  └──────────┘        │
│                                                    │
│  ┌────────────────── shared ─────────────────────┐ │
│  │  code (text section)                          │ │
│  │  data (global variables)                      │ │  ← ใช้ร่วมกัน!
│  │  heap (malloc)                                │ │
│  │  open files                                   │ │
│  └───────────────────────────────────────────────┘ │
└────────────────────────────────────────────────────┘
```

แต่ละ thread มี **ของตัวเอง**:
- **Program counter (PC)** — ชี้ว่ากำลัง execute คำสั่งไหน
- **Registers** — ค่าใน CPU register
- **Stack** — local variables, function calls

แต่ thread ทุกตัว **share กัน**:
- **Code** — โค้ดโปรแกรมชุดเดียวกัน
- **Data** — global variables ตัวเดียวกัน
- **Heap** — memory ที่ malloc มาใช้ร่วมกัน
- **Open files** — ไฟล์ที่เปิดอยู่

> **จุดสำคัญ:** การที่ thread share data กัน ทำให้สื่อสารง่ายมาก (แค่อ่าน/เขียน global variable) แต่ก็ทำให้เกิดปัญหาได้ถ้าหลาย thread แก้ข้อมูลพร้อมกัน (จะเรียนเพิ่มใน Chapter 6-7)

### Thread vs Process — เปรียบเทียบ

| | Process (Lab 5) | Thread (Lab 6) |
|---|---|---|
| **สร้าง** | หนัก — ต้อง copy memory ทั้งหมด | **เบามาก** — share memory กัน |
| **Memory** | แยกกันคนละ address space | **share** address space เดียวกัน |
| **สื่อสาร** | ต้องใช้ IPC (pipe, shared memory) | อ่าน/เขียน global variable ตรง ๆ |
| **Context switch** | ช้า (ต้อง switch page table) | **เร็ว** (address space เดิม) |
| **ถ้า crash** | ไม่กระทบ process อื่น | อาจทำให้ **ทั้ง process crash** |
| **ตัวอย่าง** | Chrome แต่ละ tab = process แยก | Word: UI + spell check + auto-save |

> **เปรียบเทียบง่าย ๆ:**
> - **Process** เหมือน **บ้านแยกหลัง** — แต่ละหลังมีห้องครัว ห้องนอน ห้องน้ำของตัวเอง จะส่งของให้กันต้องส่งพัสดุ (IPC)
> - **Thread** เหมือน **คนหลายคนอยู่บ้านเดียวกัน** — ใช้ห้องครัว ห้องนั่งเล่นร่วมกัน แต่มีเตียงนอน (stack) ของตัวเอง ส่งของให้กันแค่วางไว้บนโต๊ะ (shared variable)

---

## Part 1 — สังเกต Thread ในระบบจริง

### 1.1 ดู thread ของ process จริงด้วย `ps`

ปกติ `ps aux` แสดง process — แต่ถ้าเพิ่ม `-eLf` จะเห็น **thread** ทุกตัวด้วย:

```bash
# แสดง thread ทั้งหมดในระบบ
ps -eLf | head -20
```

| Column | ความหมาย |
|---|---|
| `PID` | Process ID |
| `LWP` | Light Weight Process = **Thread ID** |
| `NLWP` | จำนวน thread ทั้งหมดของ process นี้ |

ลองดู process ที่มี thread เยอะ:

```bash
# เรียงตาม NLWP (จำนวน thread) มากสุดก่อน
ps -eLf | awk '{print $2, $6, $NF}' | sort -t' ' -k2 -rn | uniq -f1 | head -10
```

> **คำถาม 1.1:** process ไหนในระบบมีจำนวน thread มากที่สุด? มีกี่ thread?
>
> ```
> ตอบ:
>
>
> ```

### 1.2 ดู thread ของ process ที่กำลังรัน

ลองเปิด Firefox หรือโปรแกรมอื่นแล้วดู thread:

```bash
# ดู thread ของ bash ปัจจุบัน
ps -T -p $$
```

```bash
# ดู thread ผ่าน /proc
ls /proc/$$/task/
```

> **สังเกต:** `/proc/<PID>/task/` แสดง **thread ทุกตัว** ของ process — แต่ละ thread มีโฟลเดอร์ย่อยเป็น Thread ID

> **คำถาม 1.2:** bash มีกี่ thread? (ดูจาก `/proc/$$/task/`) thread ส่วนใหญ่ของโปรแกรมทั่วไปจะมีกี่ตัว?
>
> ```
> ตอบ:
>
>
> ```

### 1.3 ใช้ `htop` ดู thread แบบ real-time

```bash
htop
```

กด `H` (Shift+H) เพื่อ toggle การแสดง **thread แต่ละตัว** — ถ้าเปิดจะเห็น thread แยกบรรทัด ถ้าปิดจะเห็นแค่ process

> **สังเกต:** เมื่อกด `H` จำนวนบรรทัดจะเพิ่มขึ้นมาก — เพราะแต่ละ process อาจมีหลาย thread

กด `q` เพื่อออก

> **คำถาม 1.3:** เมื่อกด `H` ใน htop จำนวน task เปลี่ยนจากเท่าไหร่เป็นเท่าไหร่? ค่าที่เพิ่มขึ้นคือจำนวนอะไร?
>
> ```
> ตอบ:
>
>
> ```

### 1.4 สร้าง thread จาก snippet เล็ก ๆ แล้วสังเกต

สร้างไฟล์ `thread_observe.c` — โปรแกรมนี้สร้าง 3 thread แล้ว sleep ให้เราดู:

```c
#include <stdio.h>
#include <pthread.h>
#include <unistd.h>

void *worker(void *arg) {
    int id = *(int *)arg;
    printf("Thread %d started (tid=%lu)\n", id, pthread_self());
    sleep(30);  /* นอนรอให้เราสังเกตด้วย ps */
    return NULL;
}

int main() {
    pthread_t t[3];
    int ids[] = {1, 2, 3};

    printf("PID = %d, creating 3 threads...\n", getpid());
    for (int i = 0; i < 3; i++)
        pthread_create(&t[i], NULL, worker, &ids[i]);

    printf("Check threads: ps -T -p %d\n", getpid());
    printf("Or: ls /proc/%d/task/\n", getpid());

    for (int i = 0; i < 3; i++)
        pthread_join(t[i], NULL);
    return 0;
}
```

```bash
gcc -Wall thread_observe.c -o thread_observe -lpthread
./thread_observe &
```

ขณะที่โปรแกรมทำงาน (30 วินาที) ลองใช้เครื่องมือต่าง ๆ ดู:

```bash
# ดู thread ด้วย ps
ps -T -p $(pgrep thread_observe)

# ดู thread ผ่าน /proc
ls /proc/$(pgrep thread_observe)/task/

# ดู status ของแต่ละ thread
for t in /proc/$(pgrep thread_observe)/task/*/status; do
    echo "=== $t ==="
    grep -E "^(Name|State|Pid|Tgid)" "$t"
    echo
done
```

> **สังเกต:**
> - `ps -T` แสดง 4 บรรทัด: 1 main thread + 3 worker thread
> - ทุก thread มี **PID เดียวกัน** (= TGID) แต่มี **SPID/LWP ต่างกัน** (= Thread ID)
> - ใน `/proc/<PID>/task/` มี 4 โฟลเดอร์

> **คำถาม 1.4:** จากผลลัพธ์ของ `ps -T` thread ทั้ง 4 ตัวมี PID เหมือนกันหรือไม่? SPID ต่างกันหรือไม่? สิ่งนี้บอกอะไรเกี่ยวกับ thread?
>
> ```
> ตอบ:
>
>
> ```

> **คำถาม 1.5:** ใน `/proc/<PID>/task/` แต่ละ thread มีฟิลด์ `Tgid` (Thread Group ID) เหมือนกันหรือไม่? `Tgid` คืออะไร?
>
> ```
> ตอบ:
>
>
> ```

---

## Part 2 — Thread Share Memory กัน (พิสูจน์!)

### 2.1 ทบทวน: fork() แยก memory

ใน Lab 5 เราเห็นแล้วว่า `fork()` ให้ child ได้ **copy** ของ memory:

```
  fork(): parent แก้ x = 30 → child เห็น x = 10 (copy เดิม)
```

### 2.2 thread ใช้ memory ร่วมกัน

สร้างไฟล์ `thread_shared.c`:

```c
#include <stdio.h>
#include <pthread.h>
#include <unistd.h>

int counter = 0;  /* global variable — ทุก thread เห็นตัวเดียวกัน */

void *add_five(void *arg) {
    for (int i = 0; i < 5; i++) {
        counter++;
        printf("  Thread: counter = %d\n", counter);
    }
    return NULL;
}

int main() {
    printf("Before: counter = %d\n", counter);

    pthread_t t;
    pthread_create(&t, NULL, add_five, NULL);
    pthread_join(t, NULL);

    printf("After:  counter = %d\n", counter);
    return 0;
}
```

```bash
gcc -Wall thread_shared.c -o thread_shared -lpthread
./thread_shared
```

ตัวอย่างผลลัพธ์:

```
Before: counter = 0
  Thread: counter = 1
  Thread: counter = 2
  Thread: counter = 3
  Thread: counter = 4
  Thread: counter = 5
After:  counter = 5
```

> **สังเกต:** thread เปลี่ยน counter จาก 0 เป็น 5 แล้ว main ก็เห็น counter = 5 ด้วย! **ต่างจาก fork() ที่ parent จะเห็น counter = 0**

> **คำถาม 2.1:** ถ้าใช้ `fork()` แทน `pthread_create()` ค่า counter ของ parent จะเป็นเท่าไหร่หลัง child เพิ่มเป็น 5? ทำไม?
>
> ```
> ตอบ:
>
>
> ```

### 2.3 ดู memory map เปรียบเทียบ

ลองสังเกตว่า thread ทุกตัว share address space เดียวกัน:

```bash
# สร้าง thread_observe ที่รัน background อยู่ (จาก Part 1)
./thread_observe &
PID=$!

# ดู maps ของ main thread
cat /proc/$PID/task/$(ls /proc/$PID/task/ | head -1)/maps | head -5

# ดู maps ของ thread อื่น
cat /proc/$PID/task/$(ls /proc/$PID/task/ | tail -1)/maps | head -5
```

> **สังเกต:** maps ของทุก thread **เหมือนกัน** — เพราะใช้ address space เดียวกัน!

> **คำถาม 2.2:** memory map ของ thread ต่าง ๆ ในโปรแกรมเดียวกันเหมือนหรือต่างกัน? ทำไม?
>
> ```
> ตอบ:
>
>
> ```

### 2.4 สังเกตว่า thread ทำงานพร้อมกัน — ลำดับไม่แน่นอน

สร้างไฟล์ `thread_order.c`:

```c
#include <stdio.h>
#include <pthread.h>

void *print_id(void *arg) {
    int id = *(int *)arg;
    for (int i = 0; i < 3; i++)
        printf("Thread %d: step %d\n", id, i + 1);
    return NULL;
}

int main() {
    pthread_t t[3];
    int ids[] = {1, 2, 3};

    for (int i = 0; i < 3; i++)
        pthread_create(&t[i], NULL, print_id, &ids[i]);
    for (int i = 0; i < 3; i++)
        pthread_join(t[i], NULL);
    return 0;
}
```

```bash
gcc -Wall thread_order.c -o thread_order -lpthread
```

**ลองรัน 3 ครั้ง** แล้วเปรียบเทียบ:

```bash
./thread_order
echo "---"
./thread_order
echo "---"
./thread_order
```

> **สังเกต:** ลำดับ output **ไม่เหมือนกันทุกครั้ง**! เพราะ OS เป็นคน schedule ว่า thread ไหนจะได้ทำงานเมื่อไหร่

> **คำถาม 2.3:** ลำดับการพิมพ์เหมือนกันทุกครั้งหรือไม่? ทำไม OS ไม่ทำให้ลำดับเหมือนกัน?
>
> ```
> ตอบ:
>
>
> ```

---

## Part 3 — Threading Issues: fork() กับ Signal ใน Multithreaded Program

ใน Lab 5 เราเรียนเรื่อง `fork()`, `exec()` และ **signal** ไปแล้ว — แต่ตอนนั้นเป็น single-threaded program พอเป็น **multithreaded** จะมีคำถามสำคัญที่ต้องรู้:

- ถ้า process มีหลาย thread แล้ว **fork()** จะ copy ทุก thread หรือแค่ thread ที่เรียก?
- **Signal** จะถูกส่งไปที่ thread ไหน?

### 3.1 fork() ใน multithreaded program

**กฎของ Linux:** เมื่อเรียก `fork()` ใน multithreaded program **child process จะมีแค่ 1 thread** คือ thread ที่เรียก `fork()` เท่านั้น — thread อื่น ๆ **จะหายไปหมด**!

```
  Parent process (3 threads)            Child process (1 thread)
  ┌──────────────────────┐              ┌──────────────────────┐
  │  Thread 1 (main)     │   fork()     │  Thread 1 (main)     │
  │  Thread 2 (worker)   │  ───────►    │                      │
  │  Thread 3 (worker)   │              │  Thread 2, 3 หายไป!  │
  └──────────────────────┘              └──────────────────────┘
```

สร้างไฟล์ `thread_fork.c` เพื่อพิสูจน์:

```c
#include <stdio.h>
#include <stdlib.h>
#include <pthread.h>
#include <unistd.h>
#include <sys/wait.h>

void *worker(void *arg) {
    int id = *(int *)arg;
    printf("[Parent] Thread %d running (tid=%lu)\n", id, pthread_self());
    sleep(10);
    return NULL;
}

int main() {
    pthread_t t[2];
    int ids[] = {1, 2};

    /* สร้าง 2 worker threads ใน parent */
    for (int i = 0; i < 2; i++)
        pthread_create(&t[i], NULL, worker, &ids[i]);

    sleep(1);  /* รอให้ thread เริ่มทำงาน */

    printf("\n[Parent] PID=%d, threads in parent:\n", getpid());
    fflush(stdout);

    /* นับ thread ของ parent */
    char path[64];
    int count = 0;
    sprintf(path, "/proc/%d/task", getpid());
    FILE *fp = popen("ls /proc/self/task | wc -l", "r");
    if (fp) { fscanf(fp, "%d", &count); pclose(fp); }
    printf("  Thread count: %d\n", count);

    /* fork! */
    pid_t pid = fork();

    if (pid == 0) {
        /* child process */
        printf("\n[Child]  PID=%d, threads in child:\n", getpid());
        fflush(stdout);

        count = 0;
        fp = popen("ls /proc/self/task | wc -l", "r");
        if (fp) { fscanf(fp, "%d", &count); pclose(fp); }
        printf("  Thread count: %d\n", count);

        printf("[Child]  Only the thread that called fork() survives!\n");
        _exit(0);
    }

    wait(NULL);

    for (int i = 0; i < 2; i++)
        pthread_join(t[i], NULL);
    return 0;
}
```

```bash
gcc -Wall thread_fork.c -o thread_fork -lpthread
./thread_fork
```

ตัวอย่างผลลัพธ์:

```
[Parent] Thread 1 running (tid=140234567890)
[Parent] Thread 2 running (tid=140234567891)

[Parent] PID=1234, threads in parent:
  Thread count: 3

[Child]  PID=1235, threads in child:
  Thread count: 1
[Child]  Only the thread that called fork() survives!
```

> **สังเกต:** parent มี 3 thread (main + 2 workers) แต่ child มีแค่ **1 thread** เท่านั้น!

> **คำถาม 3.1:** child process มีกี่ thread? ทำไม Linux ไม่ copy ทุก thread ไปด้วย? (คิดว่าจะมีปัญหาอะไรถ้า copy ทุก thread)
>
> ```
> ตอบ:
>
>
> ```

### 3.2 exec() ใน multithreaded program

`exec()` จะ **replace ทั้ง process** — ทุก thread หายไปหมด ไม่ว่าจะมีกี่ thread ก็ถูกแทนที่ด้วยโปรแกรมใหม่

> **กฎ:** fork() copy แค่ 1 thread, exec() ลบทุก thread แล้วเริ่มใหม่

ลองดูด้วย strace ว่า fork กับ thread creation ใช้ system call อะไรบ้าง:

```bash
# ดู strace ของ fork+exec ใน multithreaded program
strace -f -e trace=clone3,clone,execve ./thread_fork 2>&1 | grep -E "clone|exec"
```

> **สังเกต:** จะเห็น `clone3(...)` ตอนสร้าง thread และ `clone(...)` หรือ `clone3(...)` ตอน fork — ทั้งคู่ใช้ clone เป็น base แต่ flags ต่างกัน (จะเรียนใน Part 4)

> **คำถาม 3.2:** ถ้า multithreaded web server ต้อง fork+exec เพื่อรัน CGI script จะเกิดอะไรกับ worker threads อื่น ๆ ใน child?
>
> ```
> ตอบ:
>
>
> ```

### 3.3 Signal ใน multithreaded program

ใน Lab 5 เราเรียนว่า signal ถูกส่งไปที่ **process** — แต่ถ้า process มีหลาย thread **thread ไหนจะรับ signal?**

**กฎของ Linux:**
- **Synchronous signals** (เช่น SIGSEGV, SIGFPE) → ส่งไปที่ **thread ที่ทำให้เกิด**
- **Asynchronous signals** (เช่น SIGINT, SIGTERM) → ส่งไปที่ **thread ใดก็ได้** ที่ไม่ได้ block signal นั้น

สร้างไฟล์ `thread_signal.c`:

```c
#include <stdio.h>
#include <pthread.h>
#include <signal.h>
#include <unistd.h>

void handler(int sig) {
    printf("  Signal %d received by thread %lu\n", sig, pthread_self());
}

void *worker(void *arg) {
    int id = *(int *)arg;
    printf("Thread %d: tid=%lu\n", id, pthread_self());

    /* ทำงานวน loop */
    for (int i = 0; i < 30; i++) {
        usleep(500000);  /* 0.5 วินาที */
    }
    return NULL;
}

int main() {
    /* ตั้ง handler สำหรับ SIGUSR1 */
    struct sigaction sa = { .sa_handler = handler };
    sigaction(SIGUSR1, &sa, NULL);

    printf("Main thread: tid=%lu\n", pthread_self());
    printf("PID=%d — ส่ง signal ด้วย: kill -USR1 %d\n\n", getpid(), getpid());

    pthread_t t[3];
    int ids[] = {1, 2, 3};

    for (int i = 0; i < 3; i++)
        pthread_create(&t[i], NULL, worker, &ids[i]);

    /* รอให้ thread เริ่มแล้วบอก user ให้ส่ง signal */
    sleep(1);
    printf("\nNow send signals: kill -USR1 %d\n", getpid());
    printf("Try sending multiple times and see which thread receives it\n\n");

    for (int i = 0; i < 3; i++)
        pthread_join(t[i], NULL);
    return 0;
}
```

```bash
gcc -Wall thread_signal.c -o thread_signal -lpthread
./thread_signal &
```

**เปิดอีก terminal แล้วส่ง signal หลายครั้ง:**

```bash
# ส่ง SIGUSR1 ไป 5 ครั้ง (แทน <PID> ด้วย PID จริง)
for i in 1 2 3 4 5; do kill -USR1 $(pgrep thread_signal); sleep 1; done
```

> **สังเกต:** แต่ละครั้ง signal อาจถูกรับโดย **thread คนละตัว** — OS เลือกเอง!

> **คำถาม 3.3:** signal ถูกรับโดย thread เดียวกันทุกครั้งหรือไม่? ทำไม? (อ้างอิงจากกฎที่เรียนไป)
>
> ```
> ตอบ:
>
>
> ```

### 3.4 ส่ง signal ไปที่ thread เฉพาะด้วย `pthread_kill()`

ปกติ `kill()` ส่ง signal ไปที่ **process** (OS เลือก thread ให้) แต่ `pthread_kill()` ส่งไปที่ **thread ที่ระบุ** ได้:

```c
pthread_kill(thread_id, SIGUSR1);  /* ส่ง SIGUSR1 ไปที่ thread ที่ระบุ */
```

| API | ส่ง signal ไปที่ | ใครรับ? |
|---|---|---|
| `kill(pid, sig)` | Process | thread ใดก็ได้ (OS เลือก) |
| `pthread_kill(tid, sig)` | Thread ที่ระบุ | thread ที่ระบุเท่านั้น |

> **คำถาม 3.4:** ถ้าต้องการให้ main thread เป็นคนรับ signal ทุกครั้ง มีวิธีไหนบ้าง? (hint: มี 2 วิธี)
>
> ```
> ตอบ:
>
>
> ```

---

## Part 4 — Linux Threads ภายใน: clone() System Call

ในสไลด์เราเรียนว่า Linux ไม่ได้แยก "process" กับ "thread" จริง ๆ — ภายใน kernel ทุกอย่างเป็น **task** ที่สร้างด้วย `clone()` system call ตัวเดียวกัน! สิ่งที่ต่างคือ **flags** ที่บอกว่าจะ share อะไรบ้าง

### 4.1 เปรียบเทียบ clone() flags ของ fork vs thread

| Flag | ความหมาย | fork() | pthread_create() |
|---|---|---|---|
| `CLONE_VM` | share memory space | ไม่ใช้ (copy memory) | **ใช้** |
| `CLONE_FILES` | share open files | ไม่ใช้ (copy file table) | **ใช้** |
| `CLONE_SIGHAND` | share signal handlers | ไม่ใช้ (copy) | **ใช้** |
| `CLONE_FS` | share filesystem info | ไม่ใช้ (copy) | **ใช้** |
| `CLONE_THREAD` | อยู่ใน thread group เดียวกัน | ไม่ใช้ | **ใช้** |

> **สรุป:** fork() = clone() **ไม่ share** อะไรเลย, pthread_create() = clone() **share ทุกอย่าง**

### 4.2 strace ดู clone() จริง — fork vs thread

ลองใช้โปรแกรมจาก Part 1 (`thread_observe`) เปรียบเทียบกับ fork:

```bash
# ดู clone flags ตอนสร้าง thread
strace -f -e trace=clone3,clone ./thread_observe 2>&1 | head -10
```

ตัวอย่างผลลัพธ์:

```
clone3({flags=CLONE_VM|CLONE_FS|CLONE_FILES|CLONE_SIGHAND|CLONE_THREAD|...}, ...) = 1235
clone3({flags=CLONE_VM|CLONE_FS|CLONE_FILES|CLONE_SIGHAND|CLONE_THREAD|...}, ...) = 1236
clone3({flags=CLONE_VM|CLONE_FS|CLONE_FILES|CLONE_SIGHAND|CLONE_THREAD|...}, ...) = 1237
```

> **สังเกต:** `pthread_create()` ภายในเรียก `clone3()` พร้อม flags `CLONE_VM|CLONE_FS|CLONE_FILES|CLONE_SIGHAND|CLONE_THREAD` — share ทุกอย่าง!

ลองเปรียบเทียบกับ fork:

```bash
# สร้างโปรแกรมง่าย ๆ ที่ fork
echo '#include <unistd.h>
#include <sys/wait.h>
int main() { if(fork()==0) _exit(0); wait(0); return 0; }' > fork_test.c
gcc fork_test.c -o fork_test

strace -f -e trace=clone3,clone ./fork_test 2>&1 | head -5
```

> **สังเกต:** fork() ใช้ `clone3()` หรือ `clone()` เช่นกัน **แต่ flags ต่างกัน** — ไม่มี `CLONE_VM`, `CLONE_THREAD`

> **คำถาม 4.1:** flags ของ `clone()` ตอนสร้าง thread ต่างจาก fork() อย่างไร? flag ไหนที่ทำให้ thread share memory กัน?
>
> ```
> ตอบ:
>
>
> ```

### 4.3 ดู task_struct ผ่าน /proc — thread กับ process ต่างกันอย่างไร?

ใน kernel ทุก thread มี `task_struct` ของตัวเอง — แต่ thread ใน process เดียวกันจะ **ชี้ไปที่ data structure เดียวกัน** (เช่น memory map, file table)

ลองเปรียบเทียบข้อมูลของ thread ใน process เดียวกัน:

```bash
# รัน thread_observe อยู่ background (จาก Part 1)
./thread_observe &
PID=$!

echo "=== เปรียบเทียบ thread ใน process เดียวกัน ==="
for tid in $(ls /proc/$PID/task/); do
    echo "--- TID: $tid ---"
    # ดู PID, TGID (Thread Group ID), และ VmSize (memory)
    grep -E "^(Pid|Tgid|VmSize|Threads)" /proc/$PID/task/$tid/status
    echo
done
```

> **สังเกต:**
> - ทุก thread มี `Tgid` (Thread Group ID) เหมือนกัน = PID ของ process
> - ทุก thread มี `VmSize` เท่ากัน — เพราะ share address space เดียวกัน
> - แต่ `Pid` (ในที่นี้คือ TID) ต่างกัน

ลองดู file descriptors ที่ share กัน:

```bash
# ดู file descriptors — ทุก thread เห็นเหมือนกัน
echo "=== FD ของ thread ต่าง ๆ ==="
FIRST_TID=$(ls /proc/$PID/task/ | head -1)
LAST_TID=$(ls /proc/$PID/task/ | tail -1)
echo "Thread $FIRST_TID:" && ls /proc/$PID/task/$FIRST_TID/fd
echo "Thread $LAST_TID:" && ls /proc/$PID/task/$LAST_TID/fd
```

> **สังเกต:** file descriptors ของทุก thread เหมือนกัน — เพราะ `CLONE_FILES` ทำให้ share file table

> **คำถาม 4.2:** ทำไม thread ทุกตัวถึงมี `VmSize` เท่ากัน? แล้ว `Tgid` กับ `Pid` ต่างกันอย่างไร?
>
> ```
> ตอบ:
>
>
> ```

### 4.4 Concurrency vs Parallelism — สังเกตจริงบนเครื่อง

**Concurrency** = หลาย task ทำงาน "สลับกัน" (interleaving) — แม้มี CPU เดียวก็ทำได้
**Parallelism** = หลาย task ทำงาน "พร้อมกันจริง ๆ" — ต้องมีหลาย CPU cores

```bash
# ดูจำนวน CPU cores
nproc
```

ลองสร้างโปรแกรมที่ใช้หลาย thread แล้วดูว่าทำงานบน core ไหน:

```c
/* thread_cores.c */
#define _GNU_SOURCE
#include <stdio.h>
#include <pthread.h>
#include <sched.h>
#include <unistd.h>

void *worker(void *arg) {
    int id = *(int *)arg;
    for (int i = 0; i < 5; i++) {
        printf("Thread %d running on CPU core %d\n", id, sched_getcpu());
        usleep(100000);
    }
    return NULL;
}

int main() {
    printf("System has %d CPU cores\n\n", (int)sysconf(_SC_NPROCESSORS_ONLN));

    pthread_t t[4];
    int ids[] = {1, 2, 3, 4};

    for (int i = 0; i < 4; i++)
        pthread_create(&t[i], NULL, worker, &ids[i]);
    for (int i = 0; i < 4; i++)
        pthread_join(t[i], NULL);
    return 0;
}
```

```bash
gcc -Wall thread_cores.c -o thread_cores -lpthread
./thread_cores
```

> **สังเกต:** thread แต่ละตัวอาจทำงานบน **core ต่างกัน** (parallelism) หรือ **core เดียวกัน** (concurrency) — ขึ้นอยู่กับ OS scheduler และจำนวน cores

> **คำถาม 4.3:** thread ทั้ง 4 ตัวทำงานบน core เดียวกันหรือต่างกัน? ถ้าเครื่องมีแค่ 1 core จะเกิดอะไรขึ้น?
>
> ```
> ตอบ:
>
>
> ```

> **คำถาม 4.4:** อธิบายความแตกต่างระหว่าง concurrency กับ parallelism ด้วยคำพูดของตัวเอง
>
> ```
> ตอบ:
>
>
> ```

---

## Part 5 — Thread vs Process: เปรียบเทียบจากการสังเกต

### 5.1 Benchmark — สร้าง thread เร็วกว่า fork กี่เท่า?

สร้างไฟล์ `bench.c`:

```c
#include <stdio.h>
#include <unistd.h>
#include <pthread.h>
#include <sys/wait.h>
#include <time.h>

#define COUNT 5000

void *noop(void *arg) { return NULL; }

int main() {
    struct timespec s, e;

    /* Benchmark: thread */
    clock_gettime(CLOCK_MONOTONIC, &s);
    for (int i = 0; i < COUNT; i++) {
        pthread_t t;
        pthread_create(&t, NULL, noop, NULL);
        pthread_join(t, NULL);
    }
    clock_gettime(CLOCK_MONOTONIC, &e);
    double thread_time = (e.tv_sec - s.tv_sec) + (e.tv_nsec - s.tv_nsec) / 1e9;

    /* Benchmark: fork */
    clock_gettime(CLOCK_MONOTONIC, &s);
    for (int i = 0; i < COUNT; i++) {
        if (fork() == 0) _exit(0);
        else wait(NULL);
    }
    clock_gettime(CLOCK_MONOTONIC, &e);
    double fork_time = (e.tv_sec - s.tv_sec) + (e.tv_nsec - s.tv_nsec) / 1e9;

    printf("Thread: %d create+join in %.3f sec (%.0f us each)\n",
           COUNT, thread_time, thread_time / COUNT * 1e6);
    printf("Fork:   %d fork+wait  in %.3f sec (%.0f us each)\n",
           COUNT, fork_time, fork_time / COUNT * 1e6);
    printf("Fork is %.1fx slower than thread\n", fork_time / thread_time);
    return 0;
}
```

```bash
gcc -Wall bench.c -o bench -lpthread
./bench
```

ตัวอย่างผลลัพธ์:

```
Thread: 5000 create+join in 0.250 sec (50 us each)
Fork:   5000 fork+wait  in 1.500 sec (300 us each)
Fork is 6.0x slower than thread
```

> **คำถาม 5.1:** thread สร้างเร็วกว่า fork กี่เท่า? ทำไม? (อธิบายจากมุมมอง memory)
>
> ```
> ตอบ:
>
>
> ```

### 5.2 สรุป — เมื่อไหร่ใช้อะไร?

| สถานการณ์ | ใช้ Thread | ใช้ Process (fork) |
|---|---|---|
| งานที่ต้อง share ข้อมูลมาก | **เหมาะ** (share memory ง่าย) | ไม่เหมาะ (ต้องใช้ IPC) |
| งานที่ต้องการ isolation | ไม่เหมาะ (crash ลาม) | **เหมาะ** (crash ไม่ลาม) |
| งานที่สร้าง/ทำลายบ่อย | **เหมาะ** (เร็ว) | ไม่เหมาะ (ช้า) |
| Web server | Worker threads | Multi-process (Apache prefork) |
| Web browser tabs | ไม่เหมาะ (1 tab crash ลามทุก tab) | **เหมาะ** (Chrome ใช้วิธีนี้) |

> **คำถาม 5.2:** ถึงแม้ thread จะเร็วกว่า แต่มีกรณีไหนบ้างที่ควรใช้ process (fork) แทน? (ยกตัวอย่างอย่างน้อย 2 กรณี)
>
> ```
> ตอบ:
>
>
> ```

---

## สรุปคำสั่งที่ใช้ในแลป

### คำสั่ง Linux สำหรับดู Thread

| คำสั่ง | หน้าที่ |
|---|---|
| `ps -eLf` | แสดง thread ทั้งหมดในระบบ |
| `ps -T -p <PID>` | แสดง thread ของ process ที่ระบุ |
| `htop` + กด `H` | ดู thread แบบ real-time |
| `/proc/<PID>/task/` | โฟลเดอร์ของ thread แต่ละตัว |
| `strace -f -e trace=clone3,clone` | ดู clone() system call ตอนสร้าง thread/process |
| `nproc` | ดูจำนวน CPU cores |
| `sched_getcpu()` | ดูว่า thread ทำงานบน core ไหน |

### Pthreads API

| API | หน้าที่ | เปรียบเทียบกับ Process |
|---|---|---|
| `pthread_create(&tid, NULL, func, arg)` | สร้าง thread ใหม่ | เหมือน `fork()` |
| `pthread_join(tid, &retval)` | รอ thread ทำงานเสร็จ | เหมือน `wait()` |
| `pthread_kill(tid, sig)` | ส่ง signal ไปที่ thread ที่ระบุ | เหมือน `kill(pid, sig)` |
| `pthread_self()` | ดู thread ID ของตัวเอง | เหมือน `getpid()` |

### Compile Flag

```bash
gcc -Wall program.c -o program -lpthread
#                                ^^^^^^^^
#                    ต้องใส่ -lpthread ทุกครั้ง!
```

---

## สรุป

ในแลปนี้เราได้เรียนรู้:

1. **Thread** คือหน่วยการทำงานที่เบากว่า process — มี stack, registers, PC ของตัวเอง แต่ **share code, data, heap** กับ thread อื่นใน process เดียวกัน
2. ดู thread จริงในระบบได้ด้วย `ps -T`, `htop`, `/proc/<PID>/task/`
3. Thread **share memory** กัน — ต่างจาก fork() ที่แยก memory คนละชุด
4. **Threading Issues:** `fork()` ใน multithreaded program จะ copy แค่ thread ที่เรียก, `exec()` จะ replace ทั้ง process, signal อาจถูกรับโดย thread ใดก็ได้
5. **Linux ใช้ `clone()` system call** สร้างทั้ง process และ thread — ต่างกันที่ **flags** (CLONE_VM, CLONE_THREAD ฯลฯ)
6. **Concurrency** คือหลาย task สลับทำงาน, **Parallelism** คือทำงานพร้อมกันจริงบนหลาย cores
7. การสร้าง thread **เร็วกว่า** fork หลายเท่า — เพราะไม่ต้อง copy address space
8. **Thread เหมาะสำหรับงาน share ข้อมูล**, **Process เหมาะสำหรับงานที่ต้องการ isolation**
