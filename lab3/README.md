## Этап 1. Статическая библиотека

Компиляция C-кода, создание архива и компоновка с C++:

```console
$ gcc -c vector_math.c -o vector_math.o
$ ar rcs libmath_static.a vector_math.o
$ g++ -c main.cpp -o main.o
$ g++ main.o -L. -lmath_static -o app_static
$ ./app_static
Library version: v2.4.1-release
Dot product result: 32
```

`-L.` добавляет текущую директорию в пути поиска библиотек, `-lmath_static` выбирает `libmath_static.a`. Объект `vector_math.o` извлекается из архива, и его код включается в исполняемый файл. Здесь статическая компоновка относится к математической библиотеке.

### 1.1. Что делают флаги ar r, c, s? Что произойдёт при повторном запуске?

- `r` - добавляет объектные файлы в архив, заменяя существующие члены с теми же именами.
- `c` - разрешает создание архива без предупреждения о том, что его ещё нет.
- `s` - создаёт или обновляет индекс символов, который используется при компоновке.

Повторная команда заменяет `vector_math.o` в существующем архиве. Дубликат этого члена не добавляется, другие члены архива сохраняются.

```console
$ ar t libmath_static.a
vector_math.o
$ ar rcs libmath_static.a vector_math.o
$ ar t libmath_static.a
vector_math.o
```

### 1.2. Почему main не подверглась name mangling?

```console
$ nm main.o
                 U _GLOBAL_OFFSET_TABLE_
                 U _ZNSolsEPFRSoS_E
                 U _ZNSolsEi
0000000000000027 r _ZNSt8__detail30__integer_to_chars_is_unsignedIjEE
0000000000000028 r _ZNSt8__detail30__integer_to_chars_is_unsignedImEE
0000000000000029 r _ZNSt8__detail30__integer_to_chars_is_unsignedIyEE
                 U _ZSt21ios_base_library_initv
                 U _ZSt4cout
                 U _ZSt4endlIcSt11char_traitsIcEERSt13basic_ostreamIT_T0_ES6_
                 U _ZStlsISt11char_traitsIcEERSt13basic_ostreamIcT_ES5_PKc
                 U __stack_chk_fail
                 U dot_product
                 U get_lib_version
0000000000000000 T main
```

`main` - специальная функция начала выполнения C++-программы. GCC сохраняет имя `main`, на которое ссылается стартовый код среды выполнения. Она не является обычной перегружаемой функцией, для которой требуется кодировать типы аргументов в имени.

`T main` означает, что функция определена в секции кода. Символы `dot_product` и `get_lib_version` тоже имеют простые имена, но по другой причине: для них в заголовке задано связывание `extern "C"`. Буква `U` означает, что их определения предстоит найти при компоновке.

## Этап 2. Динамическая библиотека

### 2.1. Результат сборки без явного -fPIC

```console
$ gcc -c vector_math.c -o vector_math_nopic.o
$ gcc -shared -o libmath_broken.so vector_math_nopic.o
```

**Обе команды завершились успешно, код возврата - `0`, ошибок и предупреждений нет.** Несмотря на имя `libmath_broken.so`, в данном окружении файл создан корректно. Поэтому привести ожидаемую в условии ошибку для этих исходников нельзя.

В Ubuntu компилятор по умолчанию включает генерацию PIE:

```console
$ gcc -Q --help=common | grep -E 'fPIC|fPIE'
  -fPIC                               [enabled]
  -fPIE                               [enabled]
```

Кроме того, данный код обращается к локальному `static`-указателю. В полученном объектном файле доступ к нему использует относительную релокацию `R_X86_64_PC32`, а абсолютная `R_X86_64_64` относится к указателю в секции данных:

```console
$ readelf -rW vector_math_nopic.o

Relocation section '.rela.text' at offset 0x270 contains 1 entry:
    Offset             Info             Type               Symbol's Value  Symbol's Name + Addend
000000000000000b  0000000400000002 R_X86_64_PC32          0000000000000000 .data.rel.local - 4

Relocation section '.rela.data.rel.local' at offset 0x288 contains 1 entry:
    Offset             Info             Type               Symbol's Value  Symbol's Name + Addend
0000000000000000  0000000300000001 R_X86_64_64            0000000000000000 .rodata + 0

Relocation section '.rela.eh_frame' at offset 0x2a0 contains 2 entries:
    Offset             Info             Type               Symbol's Value  Symbol's Name + Addend
0000000000000020  0000000200000002 R_X86_64_PC32          0000000000000000 .text + 0
0000000000000040  0000000200000002 R_X86_64_PC32          0000000000000000 .text + 11
```

Эти релокации допустимы для полученной `.so`. `readelf -dW libmath_broken.so` не показывает `TEXTREL`. Отсутствие явно указанного `-fPIC` не гарантирует ошибку для любого исходника и настроек компилятора. Для сборки разделяемой библиотеки далее явно используется `-fPIC`.

```console
$ gcc -c -fPIC vector_math.c -o vector_math_pic.o
$ gcc -shared -o libmath_dynamic.so vector_math_pic.o
$ g++ main.o -L. -lmath_dynamic -o app_dynamic
```

`-L.` задаёт путь поиска при компоновке, но не добавляет его в путь поиска загрузчика. Поэтому обычный запуск в данном окружении завершается ошибкой с кодом `127`:

```console
$ ./app_dynamic
./app_dynamic: error while loading shared libraries: libmath_dynamic.so: cannot open shared object file: No such file or directory
```

Для запуска указываем загрузчику расположение `.so`:

```console
$ LD_LIBRARY_PATH=. ./app_dynamic
Library version: v2.4.1-release
Dot product result: 32
```

### 2.2. Сравнение text у app_static и app_dynamic

```console
$ size app_static app_dynamic
   text    data     bss     dec     hex filename
   2548     656     280    3484     d9c app_static
   2517     680     280    3477     d95 app_dynamic
```

Колонка `text` уменьшилась с **2548** до **2517 байт**, то есть на **31 байт**. Разница невелика: библиотека маленькая, а динамическая компоновка также требует служебных структур и переходов к внешним функциям. Значение `text` учитывает не только инструкции функций, но и данные только для чтения и часть метаданных.

В статическом варианте функции включены в программу, а в динамическом их код хранится в `libmath_dynamic.so` и загружается во время выполнения:

```console
$ nm app_static | grep -E 'dot_product|get_lib_version'
00000000000012ad T dot_product
000000000000129c T get_lib_version

$ nm -D app_dynamic | grep -E 'dot_product|get_lib_version'
                 U dot_product
                 U get_lib_version

$ readelf -dW app_dynamic | grep NEEDED
 0x0000000000000001 (NEEDED)             Shared library: [libmath_dynamic.so]
 0x0000000000000001 (NEEDED)             Shared library: [libstdc++.so.6]
 0x0000000000000001 (NEEDED)             Shared library: [libc.so.6]
```

## Этап 3. Порядок компоновки

### 3.1. Почему библиотека перед объектным файлом даёт undefined reference?

```console
$ g++ -L. -lmath_dynamic main.o -o app_wrong_order
/usr/bin/ld: main.o: in function `main':
main.cpp:(.text+0x63): undefined reference to `get_lib_version'
/usr/bin/ld: main.cpp:(.text+0xb4): undefined reference to `dot_product'
collect2: error: ld returned 1 exit status
```

Код возврата - `1`. Для объяснения рассмотрим символы математической библиотеки:

- `U` - ссылки, для которых пока не найдены определения.
- `D` - уже доступные определения символов.
- `E` - включённые в компоновку объектные файлы и выбранные библиотечные модули.

При правильном порядке сначала обрабатывается `main.o`: он включается в `E`, `main` добавляется в `D`, а `dot_product` и `get_lib_version` попадают в `U`. Затем библиотека предоставляет нужные определения и разрешает эти ссылки.

Для статического архива это означает извлечение подходящих `.o` из `.a` в `E`. Если архив указан первым, на тот момент соответствующих ссылок ещё нет в `U`, поэтому его член не извлекается; после появления ссылок из `main.o` автоматического возврата к архиву не происходит.

В задании используется `.so`, поэтому важна дополнительная деталь: в данном окружении g++ передаёт компоновщику **`--as-needed`**. Это подтверждается командой просмотра вызовов без выполнения сборки:

```console
$ g++ -### -L. -lmath_dynamic main.o -o app_wrong_order 2>&1 | grep -o -- --as-needed
--as-needed
```

Когда `.so` встречается до `main.o`, нужных ей неразрешённых ссылок ещё нет. С `--as-needed` библиотека не сохраняется как необходимая зависимость. Затем `main.o` добавляет ссылки в `U`, но они остаются неразрешёнными. Члены из `.so` не извлекаются по правилам архивов: здесь речь о выборе динамической зависимости.

Правильный порядок исправляет ошибку:

```console
$ g++ main.o -L. -lmath_dynamic -o app_correct
$ LD_LIBRARY_PATH=. ./app_correct
Library version: v2.4.1-release
Dot product result: 32
```

Проверка роли `--as-needed`: с его явным отключением даже обратный порядок для этой `.so` проходит успешно:

```console
$ g++ -Wl,--no-as-needed -L. -lmath_dynamic main.o -o app_no_as_needed
$ LD_LIBRARY_PATH=. ./app_no_as_needed
Library version: v2.4.1-release
Dot product result: 32
```

## Этап 4. Name mangling и extern "C"

### 4.1. Как изменились имена функций без extern "C"?

Для эксперимента в папке `no_extern_c` сохранены копии `main.cpp` и `vector_math.h`. В копии заголовка закомментированы оба блока обёртки `extern "C"`, а объявления функций оставлены прежними. Основной заголовок сохранён в рабочем состоянии.

```c
// #ifdef __cplusplus
// extern "C" {
// #endif

extern const char* get_lib_version(void);
int dot_product(const int* a, const int* b, int n);

// #ifdef __cplusplus
// }
// #endif
```

Пересобираем C++-объект с изменённым заголовком и связываем с прежней C-библиотекой:

```console
$ g++ -c no_extern_c/main.cpp -o no_extern_c/main.o
$ g++ no_extern_c/main.o -L. -lmath_dynamic -o no_extern_c/app
/usr/bin/ld: no_extern_c/main.o: in function `main':
main.cpp:(.text+0x63): undefined reference to `get_lib_version()'
/usr/bin/ld: main.cpp:(.text+0xb4): undefined reference to `dot_product(int const*, int const*, int)'
collect2: error: ld returned 1 exit status
```

Код возврата - `1`. Имена из объектного файла и библиотеки теперь не совпадают:

```console
$ nm no_extern_c/main.o | grep -E 'dot|get_lib| main$'
                 U _Z11dot_productPKiS0_i
                 U _Z15get_lib_versionv
0000000000000000 T main

$ nm libmath_dynamic.so | grep -E 'dot|get_lib'
000000000000110a T dot_product
00000000000010f9 T get_lib_version

$ nm -C no_extern_c/main.o | grep -E 'dot|get_lib'
                 U dot_product(int const*, int const*, int)
                 U get_lib_version()
```

Без `extern "C"` C++-компилятор кодирует имена по C++ ABI:

| Функция | Имя, ожидаемое C++-объектом | Имя в C-библиотеке |
| --- | --- | --- |
| `dot_product` | `_Z11dot_productPKiS0_i` | `dot_product` |
| `get_lib_version` | `_Z15get_lib_versionv` | `get_lib_version` |

Для `dot_product` закодированы параметры `const int*`, `const int*`, `int`; суффикс `v` у `get_lib_version` обозначает отсутствие параметров. Компоновщик сопоставляет имена символов, поэтому C-определения не разрешают эти C++-ссылки. `nm -C` декодирует имена только для отображения.

Обёртка `extern "C"` в основном `vector_math.h` задаёт C-связывание и обеспечивает одинаковые имена с обеих сторон. Проверки `#ifdef __cplusplus` позволяют включать один заголовок и из C, и из C++.
