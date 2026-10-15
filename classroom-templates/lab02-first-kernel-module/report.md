# Lab 02 report

- Student name: Nazar Pechkovskyi
- GitHub username: Raz1kson

Replace each placeholder with your own command output. Run HOST build commands
from this lab directory and BBB module commands from `/home/debian/labs/lab02`.
Keep the complete `vermagic`, not only its release prefix.

## Prerequisites and module metadata

### BBB: `uname -r`

```text
6.12.96-bone64
```

### HOST: `cat ~/bbb-workspace/kernel/bb-kernel/KERNEL/include/config/kernel.release`

```text
6.12.96-bone64
```

### HOST: `file lab02_hello.ko`

```text
lab02_hello.ko: ELF 32-bit LSB relocatable, ARM, EABI5 version 1 (SYSV), BuildID[sha1]=0a5f5efcd855a0cbc8f0a18c315054fd4974ad80, not stripped
```

### HOST: `modinfo lab02_hello.ko | grep vermagic`

```text
vermagic:       6.12.96-bone64 preempt mod_unload ARMv7 thumb2 p2v8 
```

Explain why the full kernel releases must match and whether yours match:

Так, версії повністю ідентичні. Якщо вони не збігаються при спробі завантаження модуля ядра з високою вірогідністю процес завершиться з помилкою "Invalid module format" або "Exec format error"

## Load, inspect, and unload on BBB

### `sudo insmod ./lab02_hello.ko name=Student` and `sudo dmesg | tail -20`

```text
debian@BeagleBone:~/labs/lab02$ sudo insmod ./lab02_hello.ko name=Student
debian@BeagleBone:~/labs/lab02$ echo $?
0
debian@BeagleBone:~/labs/lab02$ sudo dmesg | tail -20
[  399.905872] hello_module: Hello, Student, from kernel space on BBB!
```

### `lsmod | grep lab02_hello`

```text
lab02_hello            12288  0
```

### `sudo rmmod lab02_hello` and `sudo dmesg | tail -20`

```text
debian@BeagleBone:~/labs/lab02$ sudo rmmod lab02_hello
debian@BeagleBone:~/labs/lab02$ sudo dmesg | tail -20
[  399.905872] hello_module: Hello, Student, from kernel space on BBB!
[  534.962360] hello_module: Goodbye from kernel space on BBB!
```

## Parameter tests on BBB

After implementing the student tasks, rebuild, check metadata, and copy the
module again. Unload after every successful load before testing another case.
For each case record the command, result, and relevant `dmesg` output. Capture
`echo $?` immediately after `insmod` to record its exit status.

### Valid name: `name=YourName`

```text
debian@BeagleBone:~/labs/lab02$ sudo insmod ./lab02_hello.ko name=Nazar
debian@BeagleBone:~/labs/lab02$ echo $?
0
debian@BeagleBone:~/labs/lab02$ sudo dmesg | tail -20
[  618.018414] hello_module: Hello, Nazar, from kernel space on BBB!
```

### Invalid empty name: `name=` (must return `-EINVAL`)

```text
debian@BeagleBone:~/labs/lab02$ sudo insmod ./lab02_hello.ko name=
insmod: ERROR: could not insert module ./lab02_hello.ko: Invalid parameters
debian@BeagleBone:~/labs/lab02$ echo $?
1
```

### Valid count: test `count=1` and `count=10`

```text
debian@BeagleBone:~/labs/lab02$ sudo insmod ./lab02_hello.ko name=Nazar count=1
debian@BeagleBone:~/labs/lab02$ echo $?
0
debian@BeagleBone:~/labs/lab02$ sudo rmmod lab02_hello
debian@BeagleBone:~/labs/lab02$ sudo insmod ./lab02_hello.ko name=Nazar count=10
debian@BeagleBone:~/labs/lab02$ echo $?
0
debian@BeagleBone:~/labs/lab02$ sudo dmesg | tail -20
[ 1546.964087] hello_module: Hello, Nazar, from kernel space on BBB!
[ 1555.781412] hello_module: Goodbye from kernel space on BBB!
[ 1559.574233] hello_module: Hello, Nazar, from kernel space on BBB!
[ 1559.580540] hello_module: Hello, Nazar, from kernel space on BBB!
[ 1559.588774] hello_module: Hello, Nazar, from kernel space on BBB!
[ 1559.595677] hello_module: Hello, Nazar, from kernel space on BBB!
[ 1559.602393] hello_module: Hello, Nazar, from kernel space on BBB!
[ 1559.609001] hello_module: Hello, Nazar, from kernel space on BBB!
[ 1559.616363] hello_module: Hello, Nazar, from kernel space on BBB!
[ 1559.623180] hello_module: Hello, Nazar, from kernel space on BBB!
[ 1559.630476] hello_module: Hello, Nazar, from kernel space on BBB!
[ 1559.637350] hello_module: Hello, Nazar, from kernel space on BBB!
```

### Invalid count: test `count=0` and `count=11` (must return `-EINVAL`)

```text
debian@BeagleBone:~/labs/lab02$ sudo insmod ./lab02_hello.ko name=Nazar count=0
insmod: ERROR: could not insert module ./lab02_hello.ko: Invalid parameters
debian@BeagleBone:~/labs/lab02$ echo $?
1
debian@BeagleBone:~/labs/lab02$ sudo insmod ./lab02_hello.ko name=Nazar count=11
insmod: ERROR: could not insert module ./lab02_hello.ko: Invalid parameters
debian@BeagleBone:~/labs/lab02$ echo $?
1
debian@BeagleBone:~/labs/lab02$ sudo dmesg | tail -20
[ 1123.190106] hello_module: Invalid count value 0. Must be between 1 and 10
[ 1134.210262] hello_module: Invalid count value 11. Must be between 1 and 10
```

## Questions

1. Why does a kernel module not have `main()`?

Модуль ядра працює у привілейованому просторі ядра (Kernel Space), а не як звичайна програма у просторі користувача (User Space), тому він не має єдиної точки входу `main()`. Замість цього модуль є подієво-орієнтованим: у нього є окремі функції ініціалізації (викликається під час завантаження через `insmod`) та очищення (викликається під час вивантаження через `rmmod`).

2. What does `module_init()` do?

Макрос `module_init()` реєструє функцію ініціалізації модуля (наприклад, `hello_init()`), розміщуючи вказівник на неї у спеціальній секції викликів ініціалізації ядра. Ця функція автоматично виконується під час завантаження модуля командою `insmod`. Якщо функція повертає `0`, модуль завантажується успішно; якщо повертається від'ємний код помилки (наприклад, `-EINVAL`), завантаження скасовується.

3. What does `vermagic` show?
   
Рядок `vermagic` (version magic) міститься у метаданих модуля і показує точну версію ядра, під яку його було скомпільовано (початок рядка відповідає `uname -r`), цільову архітектуру (наприклад, ARMv7) та конфігураційні прапорці ядра (як-от `preempt`, `mod_unload`)

4. Why can `insmod` fail with `invalid module format`?

Помилка `invalid module format` виникає тоді, коли версія ядра у `vermagic` модуля не збігається з точним випуском запущенного ядра на платі (`uname -r`). Ядро відхиляє завантаження модуля через відсутність суворої сумісності ABI та внутрішніх структур даних.

5. What does the `name` parameter do, and what do its permissions `0444` mean?

Параметр `name` дозволяє передавати рядкове значення у модуль під час його завантаження командою `insmod lab02_hello.ko name=Student`. Права доступу `0444` (режим «тільки читання») експортують цей параметр у файлову систему `sysfs` за шляхом `/sys/module/lab02_hello/parameters/name`, дозволяючи переглядати значення через `cat`, але забороняючи змінювати його під час роботи модуля.

Submit `src/hello.c`, `Makefile`, and this completed `report.md`. Do not commit
`.ko`, `.o`, `.mod`, `.mod.c`, `.cmd`, `Module.symvers`, or `modules.order`.
