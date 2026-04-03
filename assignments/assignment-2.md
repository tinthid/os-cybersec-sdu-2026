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

**1.3)** ในแลปเราเห็นว่า keycount module ใช้ `atomic_inc()` แทน `count++` ธรรมดา

สมมุติว่ามี 2 CPU core กำลังประมวลผล keyboard interrupt พร้อมกัน — วาด **timeline** แสดงว่าถ้าใช้ `count++` ธรรมดาจะเกิดปัญหาอะไร (วาดคล้ายตาราง race condition ใน Lab 6)

จากนั้นอธิบายว่า `atomic_inc()` แก้ปัญหานี้ได้อย่างไร

> ```
> ตอบ:
>
>
> ```

**1.4)** เปรียบเทียบ 3 ช่องทางที่ kernel module ใช้สื่อสารกับ user space:

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

**2.1)** รันคำสั่งต่อไปนี้แล้ววาง output:

```bash
ps aux --sort=-%cpu | head -10
```

จาก output:
- process ที่ใช้ CPU สูงสุดคืออะไร? ทำไมถึงใช้ CPU มาก?
- process ที่มี state `S` หมายความว่ากำลังทำอะไร?
- ถ้าเห็น process ที่มี `STAT` เป็น `Ss` ตัว `s` ตัวเล็กหมายความว่าอะไร? (Hint: `man ps` แล้วดู PROCESS STATE CODES)

> ```
> ตอบ:
>
>
> ```

**2.2)** รันคำสั่งต่อไปนี้:

```bash
pstree -p $$ | head -5
```

จากนั้นอธิบาย:
- process tree ที่เห็นมี parent-child กี่ชั้น?
- `$$` คือ PID ของใคร?
- ถ้าพิมพ์ `bash` ใน terminal แล้วรัน `pstree -p $$` อีกครั้ง ผลลัพธ์จะต่างจากเดิมอย่างไร? ลองทำจริงแล้ววาง output เปรียบเทียบ

> ```
> ตอบ:
>
>
> ```

**2.3)** ในระบบจริง เราใช้ pipe `|` ทุกวัน — รันคำสั่งนี้แล้ววาง output:

```bash
strace -f -e trace=clone,pipe,dup2,execve,wait4 bash -c "cat /etc/passwd | grep root | wc -l" 2>&1
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

**2.4)** Zombie & Orphan — ทำการทดลองต่อไปนี้:

**ทดลอง A: สร้าง zombie**

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

**ทดลอง B: สร้าง orphan**

```bash
bash -c '
    echo "Parent PID = $$"
    bash -c "
        echo \"Child: my parent = \$PPID\"
        sleep 3
        echo \"Child: my parent now = \$(cat /proc/self/status | grep PPid)\"
    " &
'
sleep 4
```

วาง output ของทั้ง 2 ทดลอง แล้วตอบ:

| | Zombie | Orphan |
|---|---|---|
| เกิดจากอะไร? | | |
| ใช้ CPU/memory ไหม? | | |
| OS จัดการเองได้ไหม? | | |
| อันตรายอย่างไร? | | |
| วิธีป้องกัน? | | |

> ```
> ตอบ:
>
>
> ```

---

## ข้อ 3 — Thread & Race Condition (Lab 6)

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

**3.2)** ทดลองรันโค้ด 2 เวอร์ชัน แล้วสังเกตผลลัพธ์:

**เวอร์ชัน A — ไม่มี mutex (มี bug)**

สร้างไฟล์ `bank_bug.c`:

```c
#include <stdio.h>
#include <pthread.h>
#include <unistd.h>

int balance = 1000;

void *withdraw(void *arg) {
    int amount = *(int *)arg;
    if (balance >= amount) {
        usleep(1);  // จำลอง delay
        balance -= amount;
        printf("Withdrew %d, balance = %d\n", amount, balance);
    } else {
        printf("Insufficient funds!\n");
    }
    return NULL;
}

int main() {
    pthread_t t1, t2;
    int amt1 = 800, amt2 = 500;

    pthread_create(&t1, NULL, withdraw, &amt1);
    pthread_create(&t2, NULL, withdraw, &amt2);
    pthread_join(t1, NULL);
    pthread_join(t2, NULL);

    printf("Final balance = %d\n", balance);
    return 0;
}
```

```bash
gcc bank_bug.c -o bank_bug -lpthread
```

รัน **5 ครั้ง** แล้ววาง output ทั้ง 5 ครั้ง:

```bash
./bank_bug
./bank_bug
./bank_bug
./bank_bug
./bank_bug
```

> ```
> ตอบ (วาง output 5 ครั้ง):
>
>
> ```

**(a)** จาก output ทั้ง 5 ครั้ง — balance ติดลบบ้างไหม? ทั้ง 2 thread ถอนเงินสำเร็จทั้งคู่หรือเปล่า? (ทั้งที่เงินมีแค่ 1000 แต่ถอน 800 + 500 = 1300)

> ```
> ตอบ:
>
>
> ```

**เวอร์ชัน B — มี mutex (แก้ bug แล้ว)**

สร้างไฟล์ `bank_fixed.c`:

```c
#include <stdio.h>
#include <pthread.h>
#include <unistd.h>

int balance = 1000;
pthread_mutex_t lock = PTHREAD_MUTEX_INITIALIZER;

void *withdraw(void *arg) {
    int amount = *(int *)arg;
    pthread_mutex_lock(&lock);
    if (balance >= amount) {
        usleep(1);
        balance -= amount;
        printf("Withdrew %d, balance = %d\n", amount, balance);
    } else {
        printf("Insufficient funds!\n");
    }
    pthread_mutex_unlock(&lock);
    return NULL;
}

int main() {
    pthread_t t1, t2;
    int amt1 = 800, amt2 = 500;

    pthread_create(&t1, NULL, withdraw, &amt1);
    pthread_create(&t2, NULL, withdraw, &amt2);
    pthread_join(t1, NULL);
    pthread_join(t2, NULL);

    printf("Final balance = %d\n", balance);
    return 0;
}
```

```bash
gcc bank_fixed.c -o bank_fixed -lpthread
```

รัน **5 ครั้ง** แล้ววาง output ทั้ง 5 ครั้ง:

```bash
./bank_fixed
./bank_fixed
./bank_fixed
./bank_fixed
./bank_fixed
```

> ```
> ตอบ (วาง output 5 ครั้ง):
>
>
> ```

**(b)** เปรียบเทียบผลลัพธ์ของเวอร์ชัน A กับ B — อะไรต่างกัน? เวอร์ชัน B ยังมี balance ติดลบไหม?

> ```
> ตอบ:
>
>
> ```

**(c)** เปรียบเทียบโค้ดเวอร์ชัน A กับ B — มีบรรทัดไหนที่เพิ่มเข้ามาบ้าง? บรรทัดที่เพิ่มมาทำหน้าที่อะไร?

> ```
> ตอบ:
>
>
> ```

> ```
> ตอบ:
>
>
> ```

**3.3)** เปรียบเทียบ `fork()` กับ `pthread_create()` ในตารางด้านล่าง:

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

**5.2)** จากโค้ดด้านล่าง **trace การทำงาน** ทีละบรรทัด:

```c
int x = 0;

void *inc(void *arg) {
    for (int i = 0; i < 3; i++) x++;
    return NULL;
}

int main() {
    pthread_t t1, t2;
    pthread_create(&t1, NULL, inc, NULL);
    pthread_create(&t2, NULL, inc, NULL);
    pthread_join(t1, NULL);
    pthread_join(t2, NULL);
    printf("x = %d\n", x);
    return 0;
}
```

**(a)** ถ้าไม่มี race condition ค่า x สุดท้ายควรเป็นเท่าไหร่?

**(b)** ค่า x สุดท้ายที่ **น้อยที่สุด** ที่เป็นไปได้คือเท่าไหร่? วาด timeline (interleaving) ที่ทำให้ได้ค่านั้น
(Hint: พิจารณาว่า `x++` เป็น 3 ขั้นตอน: LOAD, ADD, STORE)

**(c)** ถ้าเพิ่ม mutex ครอบ `x++` ค่า x สุดท้ายจะเป็นเท่าไหร่เสมอ? ทำไม?

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
