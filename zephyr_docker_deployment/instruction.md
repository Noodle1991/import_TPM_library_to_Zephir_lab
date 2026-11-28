# Запуск Zephyr на OpenTitan через Docker, Verilator
### done by OguzokProduction
_______________________________


### Здесь всё ГУЧИ:

1. Первый запуск

2. Повторить проверку без сборки образа

3. Написать и запустить свою программу без GDB

4. Запустить своё приложение с OpenOCD

5. Подключить интерактивный GDB к OpenOCD

6. Запустить GDB вручную внутри отладочного контейнера

7. Получить регистры без интерактивного GDB

8. Посмотреть файловую систему Docker

9. Консоль работающего Zephyr: смотреть UART в реальном времени

10. Где смотреть ошибки

11. Краткая последовательность после первого запуска
______________________________________


Считаем, что архив скачан в `~/Downloads`

```bash
mkdir ot-zephyr-demo
unzip ~/Downloads/are_you_ready.zip -d ./ot-zephyr-demo
cd ot-zephyr-lab

chmod +x scripts/*.sh container/scripts/*.sh
```

В дальнейшем все команды `./scripts/...` выполнять из папки `ot-zephyr-lab`
________________________


### Первый запуск

```bash
./scripts/setup-and-check.sh
```

После сборки запускается пример `hello_world`, затем проверяется цепочка **Verilator → виртуальный JTAG → OpenOCD → GDB**

Если всё отстреляло и сё файн, то увидите:

<img width="700" height="411" alt="savoritor" src="https://github.com/user-attachments/assets/65609f6c-072d-4219-b0b5-d6c47776978e" />

Значения регистров `pc` и `sp` будут реальными для конкретного запуска, они могут меняться.

Проверить локальный образ:

```bash
docker image ls ot-zephyr-lab-gdb-ready
```

После проверки постоянно работающий контейнер **не остаётся**: тестовые контейнеры запускаются с `--rm`. 
Это нормально (конкретное гучи. Образ сохраняется локально в Docker Desktop → Images, А НЕ ПУБЛИКУЕТСЯ АВТОМАТИЧЕСКИ В DOCKER HUB.
______________________________


### Повторить проверку без сборки образа

Проверить, что инструменты доступны:

```bash
./scripts/check-debug-tools.sh
```

Повторить запуск `hello_world` и чтение регистров через GDB:

```bash
./scripts/smoke-test.sh
```

Логи лежат в папке `logs/`:

```text
logs/smoke_hello_world.uart0.log
logs/smoke_hello_world.sim.log
logs/smoke_hello_world.openocd.log
logs/smoke_hello_world.gdb.log
```

Посмотреть вывод Zephyr и GDB:

```bash
cat logs/smoke_hello_world.uart0.log
cat logs/smoke_hello_world.gdb.log
```
___________________________________


### Написать и запустить свою программу без GDB

Исходный файл приложения:

```text
myapp/src/main.c
```

Открыть в редакторе:

```bash
nano myapp/src/main.c
```

Например, можно заменить его содержимое на:

```c
#include <zephyr/kernel.h>

int main(void)
{
    printk("Program started\n");
    printk("Hello from OpenTitan!\n");
    return 0;
}
```

В `nano` сохранить файл сочетанием `Ctrl+O`, затем Enter, выйти через `Ctrl+X` (пометка для контуженных)

Собрать приложение:

```bash
./scripts/build-app.sh myapp hello_demo
```

Будет создан файл:

```text
build/hello_demo/zephyr/zephyr.elf
```

Запустить в Verilator **без OpenOCD и GDB**:

```bash
./scripts/run-app.sh hello_demo
```

Вывод UART появится в терминале после загрузки симулятора. Остановить запуск — `Ctrl+C` (пометка для контуженных)

Логи обычного запуска:

```bash
cat logs/hello_demo.uart0.log
cat logs/hello_demo.sim.log
```

После изменения `myapp/src/main.c` повторить две команды:

```bash
./scripts/build-app.sh myapp hello_demo
./scripts/run-app.sh hello_demo
```

**Код можно писать и без запущенного Docker**, обычным редактором на Mac. Docker потребуется для компиляции и запуска Zephyr
________________________________


### Запустить своё приложение с OpenOCD

Сначала собрать приложение, если этого ещё не делали:

```bash
./scripts/build-app.sh myapp hello_demo
```

Запустить отдельный отладочный контейнер с Verilator и OpenOCD:

```bash
./scripts/debug-app.sh hello_demo
```

Скрипт создаёт контейнер с именем `ot-zephyr-debug`, запускает симулятор,
поднимает OpenOCD и автоматически делает контрольное чтение регистров `pc` и `sp` через GDB

Если всё успешно, будет сообщение:

```text
[READY] OpenOCD GDB server is listening on port 3333 inside Docker
```

Проверить, что контейнер работает:

```bash
docker ps
```

Посмотреть сообщения отладочного контейнера:

```bash
docker logs ot-zephyr-debug
```

**Важно:** порты `3333` (GDB/OpenOCD) и `44853` (виртуальный JTAG Verilator) 
используются **внутри** контейнера. На Mac они не опубликованы. Поэтому GDB 
запускаем внутри контейнера, а не подключаем Mac к `localhost:3333`

Отладочный запуск ограничен временем: по умолчанию примерно 900 секунд. 
Это задаётся параметром `DEBUG_SIM_TIMEOUT_SECONDS`; после завершения контейнер удаляется автоматически
_______________________________


### Подключить интерактивный GDB к OpenOCD

После `./scripts/debug-app.sh hello_demo` выполнить:

```bash
./scripts/gdb-connect.sh
```

Откроется GDB внутри контейнера и подключится к OpenOCD на `localhost:3333`. В GDB можно вводить команды:

```gdb
info registers pc sp
info registers
bt
x/8i $pc
x/8wx $sp
disassemble main
continue
```

Что делают команды:

- `info registers pc sp` — показывает счётчик команд и указатель стека.
- `info registers` — показывает регистры процессора.
- `bt` — показывает стек вызовов, насколько он доступен.
- `x/8i $pc` — выводит восемь инструкций начиная с текущего адреса.
- `x/8wx $sp` — выводит восемь 32-битных слов памяти по адресу стека.
- `disassemble main` — дизассемблирует функцию `main`, если символ присутствует в ELF.
- `continue` — продолжает выполнение программы.

Если программа выполняется, остановить её в GDB можно сочетанием `Ctrl+C` (пометка для контуженных).
Для пошагового выполнения уже остановленного процессора:

```gdb
stepi
nexti
info registers pc
```

Для точки останова:

```gdb
break main
info breakpoints
```

**Уточнение:** скрипт запускает симулятор до подключения интерактивного GDB.
К моменту подключения `main()` уже может завершиться и процессор будет находиться 
в `arch_cpu_idle`. В таком случае `break main` **не сработает задним числом**. 
Для отладки старта программы потребуется перезапуск или сброс процессора с учётом 
поддержки сброса OpenOCD; в этой инструкции успешность `monitor reset halt` не предполагается

Завершить сеанс GDB:

```gdb
quit
```

Затем остановить отладочный контейнер:

```bash
./scripts/stop-debug.sh
```
______________________________


### Запустить GDB вручную внутри отладочного контейнера

Это альтернативный способ, если хочется увидеть команды подключения напрямую. 
Сначала должен работать `ot-zephyr-debug` после запуска `./scripts/debug-app.sh hello_demo`

Открыть shell **в том же контейнере**, где запущен OpenOCD:

```bash
docker exec -it ot-zephyr-debug /bin/bash
```

Внутри контейнера выполнить:

```bash
/tools/zephyr-sdk-0.16.5-1/riscv64-zephyr-elf/bin/riscv64-zephyr-elf-gdb \
  -q /workspace/build/hello_demo/zephyr/zephyr.elf
```

После приглашения `(gdb)`:

```gdb
set pagination off
target remote localhost:3333
info registers pc sp
continue
quit
```

Выйти из shell контейнера:

```bash
exit
```

Это тот же путь подключения: **GDB → OpenOCD (порт 3333) → remote_bitbang JTAG (порт 44853) → Ibex**.

Второй одновременный сеанс GDB к тому же OpenOCD не нужен: перед ручным подключением закрой предыдущий
____________________________________


### Получить регистры без интерактивного GDB

Если `ot-zephyr-debug` уже запущен:

```bash
./scripts/gdb-snapshot.sh
```

Скрипт подключится, прочитает регистры `pc` и `sp`, сохранит результат в `logs/manual.gdb.log` и отсоединится.

Посмотреть лог:

```bash
cat logs/manual.gdb.log
```
__________________________________


### Посмотреть файловую систему Docker

Для обычного shell в **новом временном контейнере**:

```bash
./scripts/shell.sh
```

Внутри Docker:

```bash
cd /home/dev/zephyrproject/zephyr
pwd
cd /home/dev/opentitan
pwd
cd /workspace
ls
exit
```

`/workspace` — это папка твоего проекта с Mac. Правки в `myapp/` через `/workspace` 
сохраняются на компьютере. А изменения произвольных файлов в 
системных папках временного контейнера при выходе не сохранятся

Если нужен shell **именно в уже запущенном отладочном контейнере**, использовать:

```bash
docker exec -it ot-zephyr-debug /bin/bash
```
____________________________________


### Где смотреть ошибки

При ошибке запуска или отладки:

```bash
cat logs/smoke_hello_world.sim.log
cat logs/smoke_hello_world.openocd.log
cat logs/smoke_hello_world.gdb.log
```

Для своего приложения в режиме отладки:

```bash
cat logs/hello_demo.debug.sim.log
cat logs/hello_demo.debug.openocd.log
cat logs/manual.gdb.log
```

Если Docker-образ уже создан, **не требуется** вручную повторять полную сборку при
каждой правке `main.c`. Достаточно `build-app.sh`, а затем `run-app.sh` или `debug-app.sh`

Если сборка из ZIP на чистой машине не может скачать зависимости (`timeout`, `unexpected EOF`,
ошибки GitHub/PyPI/Bazel), это сетевой сбой. Проверь доступ к серверам
и повтори `./scripts/setup-and-check.sh`: Docker обычно использует кэш завершённых слоёв
________________________________


### Краткая последовательность после первого запуска

**Обычная работа без GDB:**

```bash
nano myapp/src/main.c
./scripts/build-app.sh myapp hello_demo
./scripts/run-app.sh hello_demo
```

**Отладка через OpenOCD/GDB:**

```bash
./scripts/build-app.sh myapp hello_demo
./scripts/debug-app.sh hello_demo
./scripts/gdb-connect.sh
# команды вводятся уже внутри (gdb)
# после выхода из GDB:
./scripts/stop-debug.sh
```

**Проверка всего стенда:**

```bash
./scripts/smoke-test.sh
```
