# Assignment 2: ทบทวน Lab 4–6 [Due Date: TBD]
---

## ข้อ 1 — Kernel Module (Lab 4)

**1.1)** รันคำสั่งต่อไปนี้:

```bash
lsmod | head -20
```

เลือก module มา **3 ตัว** จาก output แล้วค้นหาข้อมูล (ใช้ `modinfo` หรือ Google) แล้วตอบ:

| Module | ทำหน้าที่อะไร? | เป็น driver ประเภทไหน? (device/filesystem/network/อื่นๆ) |
|---|---|---|
| 1. | | |
| 2. | | |
| 3. | | |

> ```
> ตอบ:
>
>
> ```

**1.2)** สมมุติว่า Linux ไม่มีระบบ Loadable Kernel Module — ทุกอย่างต้อง compile รวมไว้ใน kernel image ตัวเดียว:

- ถ้าต้องเพิ่ม driver ใหม่ต้องทำอย่างไร?
- ขนาดของ kernel image จะเป็นอย่างไร?
- สิ่งนี้กระทบผู้ใช้ทั่วไปอย่างไร?

> ```
> ตอบ:
>
>
> ```

**1.3)** เปรียบเทียบ 3 ช่องทางที่ kernel module ใช้สื่อสารกับ user space:

| วิธี | ตัวอย่าง | ข้อดี | ข้อจำกัด |
|---|---|---|---|
| `printk()` + `dmesg` | | | |
| `/proc` entry | | | |
| `module_param()` + `/sys` | | | |

เขียนอธิบายแต่ละช่องให้ครบ พร้อมยกตัวอย่างสถานการณ์ที่เหมาะจะใช้แต่ละวิธี

> ```
> ตอบ:
>
>
> ```

---

## ข้อ 2 — Process Observation & Analysis (Lab 5)

**2.1)** ก่อนรัน — ลองเปิดโปรแกรมให้เยอะหน่อย เช่น browser หลาย tab, เปิดเพลง, หรือรัน `stress --cpu 2 --timeout 30 &` เพื่อให้มี process ที่ใช้ CPU จริงๆ ให้วิเคราะห์

จากนั้นรันคำสั่งนี้แล้ววาง output:

```bash
ps aux --sort=-%cpu | head -10
```

> **หมายเหตุ:** ถ้า process อันดับแรกที่ใช้ CPU สูงสุดคือตัวคำสั่ง `ps` เอง ให้ข้ามไปดูอันถัดไป เพราะ `ps` จับ snapshot ตอนรันจึงเห็น CPU ของตัวเองสูงชั่วขณะ ไม่ได้สะท้อนการใช้งานจริง

จาก output:
- process ที่ใช้ CPU สูงสุด (ที่ไม่ใช่ `ps` เอง) คืออะไร? ทำไมถึงใช้ CPU มาก?
- process ที่มี state `S` หมายความว่ากำลังทำอะไร?
- ใน column `STAT` จะเห็นตัวอักษรหลายตัวต่อกัน เช่น `Ssl`, `Ss+` — ตัวแรกคือ state หลัก ตัวที่ตามมาคือ flag เพิ่มเติม ลอง `man ps` แล้วดู PROCESS STATE CODES แล้วอธิบายว่า `s`, `l`, `+` แต่ละตัวหมายความว่าอะไร?

> ```
> ตอบ:
>
>
> ```

**2.2)** รันคำสั่งต่อไปนี้:

```bash
pstree -p -s $$
```

จากนั้นอธิบาย:
- process tree ที่เห็นมี parent-child กี่ชั้น?
- `$$` คือ PID ของใคร?
- ถ้าพิมพ์ `bash` ใน terminal แล้วรัน `pstree -p -s $$` อีกครั้ง ผลลัพธ์จะต่างจากเดิมอย่างไร? ลองทำจริงแล้ววาง output เปรียบเทียบ

> ```
> ตอบ:
>
>
> ```

**2.3)** ในระบบจริง เราใช้ pipe `|` ทุกวัน — รันคำสั่งนี้แล้ววาง output:

```bash
strace -f -e trace=clone,pipe2,dup2,execve,wait4 bash -c "cat /etc/passwd | grep root | wc -l" 2>&1
```

จาก output ตอบคำถามต่อไปนี้:
- bash สร้าง pipe กี่ตัว?
- bash สร้าง child process กี่ตัว? (นับจาก `clone`)
- `dup2` ทำอะไร? ทำไม child ถึงไม่ต้องรู้จัก pipe โดยตรง?
- วาดแผนภาพแสดงการไหลของข้อมูลจาก `cat` -> `grep` -> `wc` พร้อมระบุ pipe และ file descriptor

> ```
> ตอบ:
>
>
> ```

**2.4)** Zombie Process — ทำการทดลองต่อไปนี้:

Terminal 1:
```bash
bash -c '
    bash -c "exit 0" &
    CHILD=$!
    echo "Parent=$$, Child=$CHILD"
    sleep 15
'
```

Terminal 2 (รันภายใน 15 วินาที):
```bash
ps aux | grep Z | grep -v grep
```

วาง output แล้วตอบ:
- zombie process เกิดจากอะไร?
- zombie ใช้ CPU/memory ไหม? แล้วทำไมถึงยังเป็นปัญหา?
- หลังครบ 15 วินาที ลองรัน `ps aux | grep Z` อีกครั้ง — zombie หายไปหรือไม่? เพราะอะไร?
- ในฐานะ programmer จะป้องกันไม่ให้เกิด zombie ได้อย่างไร?

> ```
> ตอบ:
>
>
> ```

---

## ข้อ 3 — Thread & Concurrency (Lab 6)

**3.1)** รันคำสั่งนี้:

```bash
ps -eLf | awk '{print $2, $6, $NF}' | sort -t' ' -k2 -rn | uniq -f1 | head -5
```

วาง output แล้วตอบ:
- process ไหนมี thread มากที่สุด?
- ทำไม process นั้นถึงต้องใช้ thread เยอะ?
- ถ้า process นั้นใช้ `fork()` สร้าง child process แทน thread จะมีข้อเสียอะไร?

> ```
> ตอบ:
>
>
> ```

**3.2)** พิจารณาโปรแกรม C ด้านล่าง:

```c
#include <stdio.h>
#include <pthread.h>
#include <unistd.h>

int shared = 0;

void *writer(void *arg) {
    for (int i = 0; i < 15; i++) {
        shared++;
        printf("[writer] shared = %d\n", shared);
        sleep(2);
    }
    return NULL;
}

void *reader(void *arg) {
    for (int i = 0; i < 15; i++) {
        printf("[reader] shared = %d\n", shared);
        sleep(2);
    }
    return NULL;
}

int main() {
    pthread_t t1, t2;
    pthread_create(&t1, NULL, writer, NULL);
    pthread_create(&t2, NULL, reader, NULL);
    printf("PID = %d — run: ps -T -p %d\n", getpid(), getpid());
    pthread_join(t1, NULL);
    pthread_join(t2, NULL);
    printf("[main] final shared = %d\n", shared);
    return 0;
}
```

Compile แล้วรัน **3 ครั้ง**:

```bash
gcc -o thread_demo thread_demo.c -pthread
./thread_demo
```

วาง output ทั้ง 3 ครั้ง แล้วตอบ:
- output แต่ละครั้งเหมือนกันหรือไม่? ถ้าไม่ เพราะอะไร?
- ค่า `shared` สุดท้ายที่ main เห็นเป็นเท่าไหร่? ทำไมค่าสุดท้ายถึงเท่ากันทุกครั้ง ทั้งที่ output ระหว่างทางต่างกัน?

> ```
> ตอบ:
>
>
> ```

**3.3)** รันโปรแกรมจากข้อ 3.2 แล้วใช้คำสั่งต่อไปนี้สังเกต (โค้ดมี `usleep()` อยู่แล้วจึงมีเวลาดู):

```bash
ps -T -p <PID>
```

วาง output แล้วตอบ:
- thread ทั้งหมดมี PID เหมือนกันหรือไม่?
- SPID (Thread ID) ต่างกันหรือไม่?
- ผลลัพธ์นี้สอดคล้องกับที่เรียนในแลปอย่างไร?

> ```
> ตอบ:
>
>
> ```

**3.4)** ใช้ `strace` ดู system call ที่โปรแกรมจากข้อ 3.2 ใช้สร้าง thread:

```bash
strace -f -e trace=clone3,clone ./thread_demo 2>&1 | head -10
```

วาง output แล้วตอบ:
- `pthread_create()` ใช้ system call อะไรภายใน?
- มี CLONE flags อะไรบ้าง? flag ไหนที่ทำให้ thread share memory กัน?

> ```
> ตอบ:
>
>
> ```

**3.5)** เปรียบเทียบ `fork()` กับ `pthread_create()` ในตารางด้านล่าง:

| หัวข้อ | `fork()` | `pthread_create()` |
|---|---|---|
| สร้างอะไร? | | |
| memory เป็นอย่างไร? | | |
| ความเร็วในการสร้าง | | |
| ถ้า crash จะกระทบอะไร? | | |
| สื่อสารกันอย่างไร? | | |
| ตัวอย่าง use case จริง | | |

> ```
> ตอบ:
>
>
> ```

---

## ข้อ 4 — Scenario Analysis (คิดวิเคราะห์)

**4.1)** สมมุติว่าคุณเป็น system admin และพบว่าเครื่อง server มีปัญหาต่อไปนี้:

> รัน `ps aux` แล้วพบ process หลายร้อยตัวที่มี state `Z` (zombie) และระบบเริ่มสร้าง process ใหม่ไม่ได้

ตอบคำถามเรียงตามลำดับ:
1. Zombie process เกิดจากอะไร?
2. ทำไมมี zombie เยอะแล้วสร้าง process ใหม่ไม่ได้?
3. คุณจะใช้คำสั่งอะไรหา **parent** ของ zombie เหล่านั้น?
4. จะแก้ปัญหาเฉพาะหน้าอย่างไร?
5. จะป้องกันไม่ให้เกิดซ้ำอย่างไร? (ด้าน programming)

> ```
> ตอบ:
>
>
> ```

**4.2)** Web server ตัวหนึ่งต้อง handle request จาก client หลายพันคนพร้อมกัน มี 2 แนวทาง:

- **แนวทาง A:** ใช้ `fork()` สร้าง child process ต่อ 1 request (เหมือน Apache prefork)
- **แนวทาง B:** ใช้ `pthread_create()` สร้าง thread ต่อ 1 request

วิเคราะห์ทั้ง 2 แนวทาง:

| หัวข้อ | แนวทาง A (fork) | แนวทาง B (thread) |
|---|---|---|
| ความเร็วในการสร้าง worker ใหม่ | | |
| การใช้ memory | | |
| ถ้า worker crash | | |
| การ share ข้อมูลระหว่าง worker | | |
| ความปลอดภัย (isolation) | | |

สรุป: จะเลือกแนวทางไหนสำหรับ web server ที่ต้อง handle หลายพัน request? เพราะอะไร?

> ```
> ตอบ:
>
>
> ```

**4.3)** พิจารณา scenario ต่อไปนี้:

> นักศึกษาเขียน kernel module ที่ `register_keyboard_notifier()` เพื่อดักจับ keystroke ทุกปุ่ม แล้วส่งข้อมูลออกไปยัง server ภายนอกผ่าน network

ตอบคำถาม:
1. Module นี้ถือว่าเป็น **keylogger** หรือไม่? ทำไม?
2. ทำไม `insmod` ถึงต้องใช้ `sudo`? ถ้าใครก็ได้โหลด module ได้จะเกิดอะไร?
3. ระบบ Linux มีกลไกอะไรบ้างในการป้องกันไม่ให้โหลด module ที่ไม่น่าเชื่อถือ? (ค้นหาเรื่อง **module signing** แล้วอธิบายสั้นๆ)
4. เปรียบเทียบ: การโจมตีผ่าน kernel module อันตรายกว่าโปรแกรม malware ธรรมดา (user space) อย่างไร?

> ```
> ตอบ:
>
>
> ```

---

## ข้อ 5 — Trace & Diagram (วาดแผนภาพ)

**5.1)** จากโค้ดด้านล่าง **วาด process tree** ที่เกิดขึ้น โดยระบุ PID (ใช้ PID สมมุติ เช่น P1, P2, ...) และระบุว่าแต่ละ process พิมพ์ output อะไร:

```c
int main() {
    fork();
    fork();
    printf("hello\n");
    return 0;
}
```

- จะมี process ทั้งหมดกี่ตัว?
- พิมพ์ "hello" กี่ครั้ง?
- วาด process tree แสดง parent-child relationship

> ```
> ตอบ:
>
>
> ```

**5.2)** สมมุติว่ามีโปรแกรม multithreaded ที่สร้าง 3 worker threads แล้วเรียก `fork()` จาก main thread

- Child process จะมีกี่ thread? อธิบายว่าทำไม
- ถ้า child เรียก `exec()` ต่อ จะเกิดอะไรกับ thread ที่เหลือ?
- วาดแผนภาพแสดงจำนวน thread ใน parent กับ child **ก่อน** และ **หลัง** fork

> ```
> ตอบ:
>
>
> ```

**5.3)** วาดแผนภาพแสดง **life cycle** ของ process ตั้งแต่ `fork()` จนถึง `exit()` โดยรวมทุกสิ่งที่เรียนมา:

แสดงให้เห็น:
1. Parent เรียก `fork()` -> เกิด child process
2. Child เรียก `exec()` -> โหลดโปรแกรมใหม่
3. Child ทำงาน -> เปลี่ยน state ระหว่าง Ready / Running / Waiting
4. Child เรียก `exit()` -> กลายเป็น zombie
5. Parent เรียก `wait()` -> zombie ถูก cleanup
6. (กรณี parent ไม่เรียก wait -> zombie ค้าง)
7. (กรณี parent ตายก่อน -> child กลายเป็น orphan -> systemd รับเลี้ยง)

วาดเป็นแผนภาพ 1 แผ่น ใช้ ASCII art หรือวาดมือแล้วถ่ายรูปแนบก็ได้

> ```
> ตอบ:
>
>
> ```

---
