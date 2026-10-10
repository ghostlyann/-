# Лабораторная: простые действия с файлами и каталогами в Linux

Все скрипты написаны на bash, проверены на GNU/Linux (bash, GNU sed, coreutils).
Перед отправкой рекомендуется проверить: `shellcheck taskNN/*`.

```
.
├── task01/task1.sh          ├── task06/check_comment.sh
├── task02/task2.sh          ├── task07/dups.sh
├── task03/banner            ├── task08/archive.sh
├── task04/idents.sh         ├── task09/spaces2tab.sh
└── task05/reg               └── task10/empty.sh
```

Сделать скрипты исполняемыми: `chmod +x task*/*`

---

## Задача 1. Отсортированный список пользователей из passwd

```bash
cut -d: -f1 /etc/passwd | sort
```

Вывод (первые строки):
```
_apt
backup
bin
claude
daemon
```

---

## Задача 2. 5 наибольших номеров из /etc/protocols

```bash
grep -v '^#' /etc/protocols | awk 'NF { print $2, $1 }' | sort -rn | head -n 5
```

Вывод (в таблице ядра/версии `protocols` значения могут отличаться):
```
262 mptcp
143 ethernet
142 rohc
141 wesp
140 shim6
```

---

## Задача 3. Программа banner

```bash
#!/bin/bash
if [ $# -ne 1 ]; then
    echo "Использование: $0 \"текст\"" >&2
    exit 1
fi

text="$1"
border=$(printf '%*s' $(( ${#text} + 2 )) '' | tr ' ' '-')

echo "+${border}+"
echo "| ${text} |"
echo "+${border}+"
```

Запуск:
```
$ ./banner "Hello from RTU MIREA!"
+-----------------------+
| Hello from RTU MIREA! |
+-----------------------+
```

---

## Задача 4. Идентификаторы в файле (без повторений)

Идентификатор по правилам C/C++/Java: `[A-Za-z_][A-Za-z0-9_]*`.

```bash
#!/bin/bash
if [ $# -ne 1 ]; then
    echo "Использование: $0 файл" >&2
    exit 1
fi

grep -oE '[A-Za-z_][A-Za-z0-9_]*' "$1" | LC_ALL=C sort -u | tr '\n' ' '
echo
```

Запуск для `hello.c`:
```c
#include <stdio.h>
int main() {
    printf("hello world\n");
    return 0;
}
```
```
$ ./idents.sh hello.c
h hello include int main n printf return stdio world
```

---

## Задача 5. Регистрация команды (reg)

```bash
#!/bin/bash
dest="${DEST:-/usr/local/bin}"

if [ $# -ne 1 ] || [ ! -f "$1" ]; then
    echo "Использование: $0 файл_программы" >&2
    exit 1
fi

chmod 755 "$1" && install -m 755 "$1" "$dest/" && echo "Установлено: $dest/$(basename "$1")"
```

Запуск (нужны права root для `/usr/local/bin`; для проверки каталог можно переопределить через `DEST`):
```
$ sudo ./reg banner
Установлено: /usr/local/bin/banner
$ ls -l /usr/local/bin/banner
-rwxr-xr-x 1 root root 294 Oct 10 10:18 /usr/local/bin/banner
```

---

## Задача 6. Комментарий в первой строке файлов .c, .js, .py

```bash
#!/bin/bash
dir="${1:-.}"

find "$dir" -type f \( -name '*.c' -o -name '*.js' -o -name '*.py' \) | sort | while IFS= read -r f; do
    first=$(head -n 1 "$f")
    case "$f" in
        *.py)      pattern='^[[:space:]]*#' ;;
        *.c|*.js)  pattern='^[[:space:]]*(//|/\*)' ;;
    esac
    if echo "$first" | grep -qE "$pattern"; then
        echo "[ЕСТЬ]  $f"
    else
        echo "[НЕТ ]  $f"
    fi
done
```

Вывод:
```
$ ./check_comment.sh demo
[ЕСТЬ]  demo/a.c
[НЕТ ]  demo/b.c
[ЕСТЬ]  demo/c.js
[ЕСТЬ]  demo/d.py
[НЕТ ]  demo/e.py
```

---

## Задача 7. Поиск дубликатов по содержимому

```bash
#!/bin/bash
dir="${1:-.}"

find "$dir" -type f -exec md5sum {} + | sort | uniq -w32 --all-repeated=separate
```

Вывод (группы дубликатов разделены пустой строкой):
```
$ ./dups.sh demo
401b30e3b8b5d629635a5c613cdb7919  demo/d.txt
401b30e3b8b5d629635a5c613cdb7919  demo/sub/e.txt

847676261680bff61c72961c8198abc0  demo/a.txt
847676261680bff61c72961c8198abc0  demo/sub/b.txt
```

---

## Задача 8. Архивация файлов по расширению в tar

```bash
#!/bin/bash
if [ $# -lt 1 ]; then
    echo "Использование: $0 расширение [каталог] [архив.tar]" >&2
    exit 1
fi

ext="${1#.}"
dir="${2:-.}"
out="${3:-archive.tar}"

find "$dir" -type f -name "*.${ext}" -print0 | tar --null -T - -cvf "$out"
echo "Создан архив: $out"
```

Вывод:
```
$ ./archive.sh log demo logs.tar
demo/a.log
demo/sub/b.log
Создан архив: logs.tar
```

---

## Задача 9. Замена 4 пробелов на табуляцию

```bash
#!/bin/bash
if [ $# -ne 2 ]; then
    echo "Использование: $0 вход выход" >&2
    exit 1
fi

sed 's/    /\t/g' "$1" > "$2"
```

Вход (`in.txt`): строки с отступами в 4 и 8 пробелов. Результат (`cat -A out.txt`, `^I` = табуляция):
```
start$
^Iindented$
^I^Itwice$
```

---

## Задача 10. Пустые текстовые файлы в каталоге

```bash
#!/bin/bash
if [ $# -ne 1 ]; then
    echo "Использование: $0 каталог" >&2
    exit 1
fi

find "$1" -type f -name '*.txt' -empty -printf '%f\n' | sort
```

Вывод (в каталоге: `empty1.txt`, `empty2.txt` — пустые, `full.txt` — с текстом, `empty.log` — пустой, но не `.txt`):
```
$ ./empty.sh demo
empty1.txt
empty2.txt
```
