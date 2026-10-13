# Лабораторная работа № 2. Задача A

Окружение: Ubuntu 24.04.1 LTS под WSL, x86-64; GCC 13.3.0, GNU Binutils 2.42.

## Этап 0. Размеры файлов и секций

### 1. Вывод команды size

```console
$ size core.o legacy.o hotfix.o main.o
   text    data     bss     dec     hex filename
    266    4040   50000   54306    d422 core.o
    172       0       4     176      b0 legacy.o
    127       0       0     127      7f hotfix.o
    118       0       0     118      76 main.o
```

### 2. Почему большая .bss почти не увеличивает файл на диске?

У `core.o` размер `.bss` - **50 000 байт**, а размер самого файла - **6200 байт**:

```console
$ stat -c '%n: %s bytes' core.o legacy.o hotfix.o main.o
core.o: 6200 bytes
legacy.o: 1608 bytes
hotfix.o: 1456 bytes
main.o: 1448 bytes

$ readelf -SW core.o | grep -E '\.data |\.bss '
  [ 3] .data             PROGBITS        0000000000000000 000088 000fc0 00  WA  0   0 32
  [ 4] .bss              NOBITS          0000000000000000 001048 00c350 00  WA  0   0 32
```

`.bss` имеет тип `NOBITS`: её содержимое не хранится на диске. В ELF есть сведения о размере и выравнивании секции, память под неё выделяется и обнуляется при загрузке программы. Здесь 50 000 байт занимает локальный символ `large_buffer`, что видно в таблице символов:

```text
     4: 0000000000000000 50000 OBJECT  LOCAL  DEFAULT    4 large_buffer
```

`dec` - это сумма `text + data + bss`. Поэтому он включает все 50 000 байт `.bss` и равен **54 306**.

## Этап 1. Извлечение строк

### 3. Какая секция со строками есть только в core.o?

Точное имя - **`.rodata`**, секция данных только для чтения:

```console
$ readelf -SW core.o legacy.o hotfix.o | grep -E '^File:|\.rodata'
File: core.o
  [ 5] .rodata           PROGBITS        0000000000000000 001048 00004c 00   A  0   0 16
File: legacy.o
File: hotfix.o
```

В `legacy.o` и `hotfix.o` такой секции нет.

### 4. Секретный токен

Все печатаемые строки, обнаруженные командой `strings -a` с минимальной длиной по умолчанию - четыре символа:

```console
$ strings -a core.o
System Initialized v1.0
FLAG{b1nary_4rch4eology_2026}
%s, Mode: %d
GCC: (Ubuntu 15.2.0-16ubuntu1) 15.2.0
core.c
large_buffer
internal_cleanup
system_mode
config_table
welcome_msg
secret_token
print_status
printf
.symtab
.strtab
.shstrtab
.rela.text
.data
.bss
.rodata
.rela.data.rel.local
.comment
.note.GNU-stack
.note.gnu.property
.rela.eh_frame
```

Токен: **`FLAG{b1nary_4rch4eology_2026}`**.

Он находится именно в `.rodata`:

```console
$ readelf -p .rodata core.o

String dump of section '.rodata':
  [     0]  System Initialized v1.0
  [    20]  FLAG{b1nary_4rch4eology_2026}
  [    3e]  %s, Mode: %d\n
```

## Этап 2. Извлечение символов

### 5. Какие имена встречаются в нескольких модулях?

```console
$ nm core.o legacy.o hotfix.o main.o

core.o:
0000000000000020 D config_table
0000000000000000 t internal_cleanup
0000000000000000 b large_buffer
0000000000000012 T print_status
                 U printf
0000000000000020 R secret_token
0000000000000000 D system_mode
0000000000000000 D welcome_msg

legacy.o:
000000000000001a T apply_legacy_patch
0000000000000000 b legacy_counter
0000000000000000 t legacy_helper
0000000000000004 C system_mode

hotfix.o:
0000000000000000 T apply_hotfix
0000000000000004 C system_mode

main.o:
                 U apply_hotfix
0000000000000000 T main
                 U print_status
```

Повторяются три имени:

- `system_mode` - в `core.o`, `legacy.o`, `hotfix.o`;
- `apply_hotfix` - в `hotfix.o` и `main.o`;
- `print_status` - в `core.o` и `main.o`.

`apply_hotfix` и `print_status` определены в одном модуле и используются другим. Аномалия - три определения `system_mode`.

### 6. Типы и секции общих символов

| Имя | Файл | Тип nm | Ndx readelf | Размещение |
| --- | --- | --- | --- | --- |
| `system_mode` | `core.o` | `D` | `3` | `.data`, размер 4 байта |
| `system_mode` | `legacy.o` | `C` | `COM` | COMMON, размер 4 байта |
| `system_mode` | `hotfix.o` | `C` | `COM` | COMMON, размер 4 байта |
| `apply_hotfix` | `hotfix.o` | `T` | `1` | `.text` |
| `apply_hotfix` | `main.o` | `U` | `UND` | Не определён в этом файле |
| `print_status` | `core.o` | `T` | `1` | `.text` |
| `print_status` | `main.o` | `U` | `UND` | Не определён в этом файле |

Подтверждение индексов и размеров:

```console
$ readelf -sW core.o legacy.o hotfix.o main.o | grep -E '^File:|system_mode|apply_hotfix|print_status'
File: core.o
     7: 0000000000000000     4 OBJECT  GLOBAL DEFAULT    3 system_mode
    11: 0000000000000012    52 FUNC    GLOBAL DEFAULT    1 print_status
File: legacy.o
     6: 0000000000000004     4 OBJECT  GLOBAL DEFAULT  COM system_mode
File: hotfix.o
     3: 0000000000000001     4 OBJECT  GLOBAL DEFAULT  COM system_mode
     4: 0000000000000000    39 FUNC    GLOBAL DEFAULT    1 apply_hotfix
File: main.o
     4: 0000000000000000     0 NOTYPE  GLOBAL DEFAULT  UND print_status
     5: 0000000000000000     0 NOTYPE  GLOBAL DEFAULT  UND apply_hotfix
```

COMMON - предварительное определение, которое компоновщик может объединить с другими COMMON-символами или заменить обычным определением. `COM` не является индексом обычной секции: место будет выбрано при компоновке.

### 7. Что можно установить о типе system_mode и в чём конфликт?

В исходном `hotfix.c` из условия `system_mode` объявлен как `char[4]`, а функция записывает байты `41 42 43 00`, то есть строку `ABC` с завершающим нулём.

В `core.o` это инициализированный 4-байтовый объект в `.data`, в `legacy.o` - 4-байтовый COMMON-объект. Только размер и тип `OBJECT` не позволяют отличить `int` от `char[4]`, поэтому дополнительно исследованы данные и инструкции:

```console
$ objdump -s -j .data --start-address=0 --stop-address=4 core.o

core.o:     file format elf64-x86-64

Contents of section .data:
 0000 01000000                             ....
```

Начальное значение в little-endian - `1`. Существенные строки вывода `objdump -dr -Mintel core.o legacy.o`:

```text
В print_status из core.o:
  1f: 8b 15 00 00 00 00       mov edx,DWORD PTR [rip+0x0]
       21: R_X86_64_PC32      system_mode-0x4

В apply_legacy_patch из legacy.o:
  27: c7 05 00 00 00 00 2a    mov DWORD PTR [rip+0x0],0x2a
  2e: 00 00 00
       29: R_X86_64_PC32      system_mode-0x8
```

`core.o` читает 32-битное значение и печатает его через `%d`, а `legacy.o` записывает 32-битное число `42`. Эти наблюдения соответствуют интерфейсу `int system_mode`.

Конфликт состоит в несовместимых типах одного внешнего объекта: число в `core.o` и `legacy.o`, массив символов в `hotfix.o`. Совпадение размера не делает типы совместимыми. Линковка может пройти успешно, но строка перезапишет байтовое представление числа.

## Этап 3. Попытка компоновки

### 8. Сообщил ли компоновщик о проблеме?

```console
$ gcc -fcommon core.o legacy.o hotfix.o main.o -o app
```

Команда завершилась с кодом `0`, без ошибок и предупреждений. Обычное определение `system_mode` из `.data` файла `core.o` имеет приоритет над COMMON-определениями. Все ссылки направляются на один объект.

В таблице символов нет полного C-типа переменной: здесь во всех случаях `OBJECT` размером 4 байта. Поэтому обычная компоновка не обнаруживает различие между `int` и `char[4]`.

Ошибка проявляется при запуске:

```console
$ ./app
System Initialized v1.0, Mode: 1
System Initialized v1.0, Mode: 4407873
```

Байты `41 42 43 00` интерпретируются как little-endian число `0x00434241 = 4407873`.

При явном включении предупреждений о COMMON-символах диагностика появляется:

```console
$ gcc -fcommon -Wl,--warn-common core.o legacy.o hotfix.o main.o -o app
/usr/bin/ld: legacy.o: warning: common of `system_mode' overridden by definition from core.o
/usr/bin/ld: hotfix.o: warning: common of `system_mode' overridden by definition from core.o
```

## Этап 4. Анализ исполняемого файла и исправление

### 9. Куда попал system_mode в исполняемом файле?

```console
$ nm app | grep system_mode
0000000000004020 D system_mode

$ readelf -sW app | grep system_mode
    33: 0000000000004020     4 OBJECT  GLOBAL DEFAULT   25 system_mode

$ readelf -SW app | grep -E '\.data |\.bss '
  [25] .data             PROGBITS        0000000000004000 003000 000fe8 00  WA  0   0 32
  [26] .bss              NOBITS          0000000000005000 003fe8 00c378 00  WA  0   0 32
```

Итоговое размещение - **`.data`**, символ типа `D`. Компоновщик выбрал обычное инициализированное определение из `core.o`, и не выделил отдельные области `.bss` для COMMON-объектов.

### 10. Что происходит с -fno-common и как исправить hotfix.c?

Команда из условия:

```console
$ gcc -fno-common main.o hotfix.o legacy.o core.o -o app_better
```

**Ошибки нет**, код возврата `0`. Запуск подтверждает, что конфликт остался:

```console
$ ./app_better
System Initialized v1.0, Mode: 1
System Initialized v1.0, Mode: 4407873
```

`-fno-common` влияет на компиляцию исходных файлов, а не преобразует символы в готовых `.o`. Поэтому для воспроизведения ожидаемой ошибки нужно пересобрать сам модуль. Исходный код из условия сохранён в `hotfix_original.c`:

```c
char system_mode[4];

void apply_hotfix(void) {
    system_mode[0] = 'A';
    system_mode[1] = 'B';
    system_mode[2] = 'C';
    system_mode[3] = '\0';
}
```

```console
$ gcc -fno-common -c hotfix_original.c -o hotfix_nocommon.o
$ nm -S hotfix_nocommon.o
0000000000000000 0000000000000027 T apply_hotfix
0000000000000000 0000000000000004 B system_mode

$ gcc -fno-common main.o hotfix_nocommon.o legacy.o core.o -o app_strict
/usr/bin/ld: warning: alignment 1 of normal symbol `system_mode' in hotfix_nocommon.o is smaller than 4 used by the common definition in legacy.o
/usr/bin/ld: warning: NOTE: alignment discrepancies can cause real problems.  Investigation is advised.
/usr/bin/ld: core.o:(.data+0x0): multiple definition of `system_mode'; hotfix_nocommon.o:(.bss+0x0): first defined here
collect2: error: ld returned 1 exit status
```

Код возврата - `1`. После пересборки `system_mode` в модуле hotfix стал обычным определением в `.bss`, которое конфликтует с определением в `.data` файла `core.o`.

Исправление без переименования переменной - оставить единственное обычное определение в `core.o`, а в `hotfix.c` использовать внешнее объявление совместимого типа и присваивать число:

```c
extern int system_mode;

void apply_hotfix(void) {
    system_mode = 42;
}
```

Здесь `42` выбрано как числовой режим, который уже записывает `apply_legacy_patch`. Просто добавить `extern` к `char system_mode[4]` недостаточно: это уберёт повторное определение, но сохранит несовместимый тип и ошибочную запись строки в число.

Исправленный исходник сохранён в `hotfix.c`. Он собирается в отдельный `hotfix_fixed.o`:

```console
$ gcc -fno-common -c hotfix.c -o hotfix_fixed.o
$ gcc -fno-common main.o hotfix_fixed.o legacy.o core.o -o app_fixed
$ nm hotfix_fixed.o
0000000000000000 T apply_hotfix
                 U system_mode

$ ./app_fixed
System Initialized v1.0, Mode: 1
System Initialized v1.0, Mode: 42
```

Компиляция, компоновка и запуск завершились с кодом `0`. В исправленном hotfix нет определения `system_mode`, только ссылка `U` на объект из `core.o`. COMMON-символ в предоставленном `legacy.o` остаётся: убрать его флагом линковки нельзя, для этого необходима пересборка самого legacy с исправленным объявлением.
