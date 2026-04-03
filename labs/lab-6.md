# Lab 6: Threads & Concurrency — สังเกต เข้าใจ ทดลอง
---

## วัตถุประสงค์

1. เข้าใจแนวคิด Thread ว่าต่างจาก Process อย่างไร
2. สังเกต thread จริงในระบบด้วย `ps`, `htop`, `/proc`
3. เห็นว่า thread **share memory** กัน (ต่างจาก process ที่แยกกัน)
4. เห็นปัญหา **race condition** ด้วยตาตัวเอง
5. เข้าใจวิธีแก้ด้วย **mutex** และ trade-off ที่ตามมา

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

> **จุดสำคัญ:** การที่ thread share data กัน ทำให้สื่อสารง่ายมาก (แค่อ่าน/เขียน global variable) แต่ก็เป็นต้นเหตุของ **race condition** ที่เราจะเจอใน Part 3

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
> - **Thread** เหมือน **คนหลายคนอยู่บ้านเดียวกัน** — ใช้ห้องครัว ห้องนั่งเล่นร่วมกัน แต่มีเตียงนอน (stack) ของตัวเอง ส่งของให้กันแค่วางไว้บนโต๊ะ (shared variable) แต่ต้องระวังอย่าใช้ห้องน้ำพร้อมกัน (race condition!)

---

## Part 1 — สังเกต Thread ในระบบจริง (30 นาที)

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

## Part 2 — Thread Share Memory กัน (พิสูจน์!) (30 นาที)

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

## Part 3 — Race Condition — ปัญหาจากการ Share Memory (30 นาที)

### ปัญหา Race Condition คืออะไร?

ใน Part 2 เราเห็นแล้วว่า thread share global variable กัน — ฟังดูสะดวกดี แต่มัน **มีปัญหาร้ายแรง** ที่ซ่อนอยู่

### ทำไม `counter++` ถึงไม่ปลอดภัย?

เราอาจคิดว่า `counter++` เป็นคำสั่งเดียว แต่ CPU จริง ๆ ทำ **3 ขั้นตอน**:

```
  C code:     counter++;

  Machine:    1. LOAD  register ← counter    (อ่านค่าจาก memory ใส่ register)
              2. ADD   register ← register + 1  (บวก 1)
              3. STORE counter ← register     (เขียนค่ากลับ memory)
```

ถ้ามี 2 thread ทำ `counter++` พร้อมกัน อาจเกิดเหตุการณ์แบบนี้:

```
  สมมุติ counter = 5

  Thread A                          Thread B
  ──────────────────               ──────────────────
  1. LOAD register_A = 5
                                   1. LOAD register_B = 5
  2. ADD  register_A = 6
                                   2. ADD  register_B = 6
  3. STORE counter = 6
                                   3. STORE counter = 6

  ผลลัพธ์: counter = 6   ← ควรเป็น 7!  (++ สองครั้ง ได้แค่ +1)
```

Thread B อ่านค่าเก่า (5) ก่อนที่ Thread A จะเขียนค่าใหม่ (6) กลับ ผลคือการ ++ ของ Thread A **หายไป**!

นี่เรียกว่า **race condition** — ผลลัพธ์ขึ้นอยู่กับ "ใครถึงก่อน" ซึ่งเปลี่ยนไปทุกครั้งที่รัน

### 3.1 เห็น race condition ด้วยตาตัวเอง

สร้างไฟล์ `race.c`:

```c
#include <stdio.h>
#include <pthread.h>

#define ITERATIONS 1000000

int counter = 0;  /* จุดเกิดเหตุ! */

void *increment(void *arg) {
    for (int i = 0; i < ITERATIONS; i++)
        counter++;
    return NULL;
}

void *decrement(void *arg) {
    for (int i = 0; i < ITERATIONS; i++)
        counter--;
    return NULL;
}

int main() {
    pthread_t t1, t2;

    printf("Expected: 0 (++ and -- cancel out)\n");

    pthread_create(&t1, NULL, increment, NULL);
    pthread_create(&t2, NULL, decrement, NULL);
    pthread_join(t1, NULL);
    pthread_join(t2, NULL);

    printf("Actual:   %d\n", counter);
    printf("Result:   %s\n", counter == 0 ? "CORRECT" : "WRONG — race condition!");
    return 0;
}
```

```bash
gcc -Wall race.c -o race -lpthread
```

**ลองรัน 5 ครั้งแล้วจดผลลัพธ์:**

```bash
for i in 1 2 3 4 5; do echo "--- Run $i ---"; ./race; done
```

ตัวอย่างผลลัพธ์:

```
Expected: 0
Actual:   -34521
Result:   WRONG — race condition!
```

> **สังเกต:** ค่าไม่เคยเป็น 0! และต่างกันทุกครั้ง! นี่คือ race condition

> **คำถาม 3.1:** รัน 5 ครั้ง จดผลลัพธ์:
>
> ```
> ตอบ:
> รอบ 1: counter =
> รอบ 2: counter =
> รอบ 3: counter =
> รอบ 4: counter =
> รอบ 5: counter =
>
> ```

### 3.2 ดู assembly จริง — พิสูจน์ว่า counter++ ไม่ใช่ atomic

```bash
gcc -S race.c -o race.s
grep -A5 "counter" race.s | head -15
```

> **สังเกต:** จะเห็น `counter++` กลายเป็นหลายคำสั่ง assembly เช่น `movl`, `addl`, `movl` (load, add, store)

> **คำถาม 3.2:** จาก assembly code ยืนยันหรือไม่ว่า `counter++` ไม่ใช่ atomic? มันเป็นกี่คำสั่ง machine?
>
> ```
> ตอบ:
>
>
> ```

### 3.3 ลดจำนวนรอบแล้วลองใหม่

ลองแก้ `ITERATIONS` เป็น 100 แล้ว compile ใหม่:

```bash
# แก้ค่าใน race.c แล้ว compile
sed 's/1000000/100/' race.c > race_small.c
gcc -Wall race_small.c -o race_small -lpthread
for i in 1 2 3 4 5; do ./race_small; done
```

> **คำถาม 3.3:** เมื่อลดจำนวนรอบเป็น 100 ผลลัพธ์มักจะเป็น 0 บ่อยขึ้นหรือไม่? ทำไม?
>
> ```
> ตอบ:
>
>
> ```

---

## Part 4 — แก้ Race Condition ด้วย Mutex (30 นาที)

### Mutex คืออะไร?

**Mutex** (Mutual Exclusion) เหมือน **กุญแจห้องน้ำ** — มีแค่ดอกเดียว ใครถือกุญแจถึงจะเข้าได้ คนอื่นต้องรอ

```
  Thread A                          Thread B
  ──────────────────                ──────────────────
  lock(mutex)   ← ได้กุญแจ!
  ┌─────────────────┐
  │ counter++       │               lock(mutex)  ← กุญแจไม่ว่าง...
  │ (critical       │                              รอ...
  │  section)       │                              รอ...
  └─────────────────┘
  unlock(mutex) ← คืนกุญแจ
                                    lock(mutex)  ← ได้กุญแจแล้ว!
                                    ┌─────────────────┐
                                    │ counter--       │
                                    │ (critical       │
                                    │  section)       │
                                    └─────────────────┘
                                    unlock(mutex)
```

**Critical section** = ส่วนของโค้ดที่ **เข้าได้ทีละ 1 thread เท่านั้น**

### 4.1 แก้ race condition ด้วย mutex

สร้างไฟล์ `race_fixed.c` (โค้ดเหมือน `race.c` แต่เพิ่ม mutex):

```c
#include <stdio.h>
#include <pthread.h>

#define ITERATIONS 1000000

int counter = 0;
pthread_mutex_t lock = PTHREAD_MUTEX_INITIALIZER;

void *increment(void *arg) {
    for (int i = 0; i < ITERATIONS; i++) {
        pthread_mutex_lock(&lock);    /* เข้า critical section */
        counter++;
        pthread_mutex_unlock(&lock);  /* ออก critical section */
    }
    return NULL;
}

void *decrement(void *arg) {
    for (int i = 0; i < ITERATIONS; i++) {
        pthread_mutex_lock(&lock);
        counter--;
        pthread_mutex_unlock(&lock);
    }
    return NULL;
}

int main() {
    pthread_t t1, t2;

    printf("Using MUTEX to protect counter\n");

    pthread_create(&t1, NULL, increment, NULL);
    pthread_create(&t2, NULL, decrement, NULL);
    pthread_join(t1, NULL);
    pthread_join(t2, NULL);

    printf("Expected: 0\n");
    printf("Actual:   %d\n", counter);
    printf("Result:   %s\n", counter == 0 ? "CORRECT — mutex works!" : "WRONG");
    return 0;
}
```

```bash
gcc -Wall race_fixed.c -o race_fixed -lpthread
```

**ลองรัน 5 ครั้ง:**

```bash
for i in 1 2 3 4 5; do echo "--- Run $i ---"; ./race_fixed; done
```

> **สังเกต:** ผลลัพธ์เป็น 0 **ทุกครั้ง**!

> **คำถาม 4.1:** ผลลัพธ์เป็น 0 ทุกครั้งหรือไม่? เปรียบเทียบกับ `race.c` ที่ไม่มี mutex
>
> ```
> ตอบ:
>
>
> ```

### 4.2 Trade-off: ถูกต้องแต่ช้าลง

Mutex ทำให้ **ถูกต้อง** แต่มี **ค่าใช้จ่าย** — เพราะ thread ต้องรอกัน ลองวัด:

```bash
echo "=== Without mutex ==="
time ./race

echo ""
echo "=== With mutex ==="
time ./race_fixed
```

> **สังเกต:** `race_fixed` **ช้ากว่า** `race` อย่างเห็นได้ชัด

> **คำถาม 4.2:** โปรแกรมที่ใช้ mutex ช้ากว่ากี่เท่า? ทำไม?
>
> ```
> ตอบ:
>
>
> ```

> **คำถาม 4.3:** นี่คือ trade-off ระหว่างอะไรกับอะไร?
>
> ```
> ตอบ:
>
>
> ```

### 4.3 Mutex API สรุป

| API | หน้าที่ | เปรียบเทียบ |
|---|---|---|
| `pthread_mutex_init(&lock, NULL)` | สร้าง mutex | เตรียมกุญแจ |
| `pthread_mutex_lock(&lock)` | lock — เข้า critical section | หยิบกุญแจ เข้าห้องน้ำ |
| `pthread_mutex_unlock(&lock)` | unlock — ออก critical section | คืนกุญแจ ออกจากห้องน้ำ |
| `pthread_mutex_destroy(&lock)` | ทำลาย mutex | ทิ้งกุญแจ |

> **กฎสำคัญ:**
> 1. ทุกครั้งที่ `lock` ต้อง `unlock` ด้วย — ถ้าลืม unlock จะ **deadlock** (thread อื่นรอตลอดกาล)
> 2. Critical section ควร **สั้นที่สุด** — อย่า lock แล้วทำงานนาน thread อื่นจะรอนาน

### 4.4 ใช้ strace ดู mutex ทำงาน

```bash
strace -f -e trace=futex ./race_fixed 2>&1 | tail -20
```

> **สังเกต:** จะเห็น `futex(...)` เยอะมาก — `futex` คือ system call ที่ Linux ใช้ implement mutex (Fast Userspace muTEX)

> **คำถาม 4.4:** `futex` system call เกี่ยวข้องกับ mutex อย่างไร?
>
> ```
> ตอบ:
>
>
> ```

---

## Part 5 — Thread vs Process: เปรียบเทียบจากการสังเกต (20 นาที)

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
| `strace -f -e trace=futex` | ดู mutex system calls |

### Pthreads API

| API | หน้าที่ | เปรียบเทียบกับ Process |
|---|---|---|
| `pthread_create(&tid, NULL, func, arg)` | สร้าง thread ใหม่ | เหมือน `fork()` |
| `pthread_join(tid, &retval)` | รอ thread ทำงานเสร็จ | เหมือน `wait()` |
| `pthread_mutex_lock(&lock)` | lock (เข้า critical section) | — |
| `pthread_mutex_unlock(&lock)` | unlock (ออก critical section) | — |

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
4. **Race condition** เกิดเมื่อหลาย thread แก้ไข shared variable พร้อมกัน — เพราะ `counter++` ไม่ใช่ atomic (เป็น 3 machine instructions)
5. **Mutex** แก้ race condition โดยให้ thread เข้า critical section ได้ทีละ 1 ตัว — แต่มี trade-off คือ **ช้าลง**
6. การสร้าง thread **เร็วกว่า** fork หลายเท่า — เพราะไม่ต้อง copy address space
7. **Thread เหมาะสำหรับงาน share ข้อมูล**, **Process เหมาะสำหรับงานที่ต้องการ isolation**
