# Кодирование файлов в текст и восстановление в Linux

## 1. Назначение

Правильное название первой кодировки — **Base64** (команда `base64`, а не `bas64`). Здесь рассматриваются Base64, классический uuencode и два менее распространённых в повседневном обмене файлами варианта: Base32 и Ascii85.

Они превращают любые байты — архив, PDF, изображение, исполняемый файл — в печатные символы. Обратное преобразование восстанавливает содержимое побайтно, если закодированные данные не повреждены. Это не шифрование и не сжатие: секретного ключа нет, размер обычно увеличивается. Декодирование не исправляет утраченные или изменённые данные.

| Формат | Где полезен | Приблизительный размер относительно исходного | Наличие в Linux |
| --- | --- | --- | --- |
| Base64 | Передача бинарных данных в текстовых полях, API, MIME | 133%, без переносов и небольшого дополнения | `base64` входит в GNU coreutils |
| uuencode | Старые Unix-системы и текстовые обменные форматы | Около 138% для длинных файлов с полными строками | `uuencode` и `uudecode` из отдельного GNU sharutils |
| Base32 | Каналы с ограниченным алфавитом; буквы A–Z и цифры 2–7 | 160%, без переносов и дополнения | `base32` входит в GNU coreutils |
| Ascii85 | Компактное текстовое представление; семейство форматов PostScript/PDF | Около 125%; нулевые блоки могут сокращаться | В инструкции используется стандартная библиотека Python |

Единого «стандартного пакета Linux» нет: состав зависит от дистрибутива и образа. В минимальном контейнере могут отсутствовать даже Python и часть coreutils. Команды ниже ориентированы на GNU/Linux. Python-скрипты рассчитаны на Python 3.6+ и не требуют `pip`.

## 2. Подготовка

> [!WARNING]
> Файл decode_file.py приведен в пункте 7 и представляет собой универсальный модуль декодирования.

Проверьте доступность команд:

```bash
command -v base64
command -v base32
command -v uuencode
command -v python3
python3 --version
```

Например, для Debian/Ubuntu необходимые пакеты устанавливаются так:

```bash
sudo apt update
sudo apt install coreutils python3
# Необязательно: только если хотите кодировать именно утилитой uuencode.
sudo apt install sharutils
```

Во всех примерах исходный файл называется `input.bin`. Замените его своим путём, например `"архив проекта.zip"`. Расширение `.bin` условное: содержимое может быть любым.

Сохраните два блока Python из разделов 7 и 8 в файлы `decode_file.py` и `encode_file.py`. Запускайте их через `python3`: права исполнения не требуются. Выходной файл Python-скриптов должен отсутствовать; для повторного запуска укажите другое имя.

**Перенаправление `>` в shell перезаписывает существующий файл.** Не указывайте одинаковый путь для входа и выхода. Для защиты от случайного затирания в текущем Bash можно выполнить `set -o noclobber`.

## 3. Base64

Кодирование утилитой:

```bash
base64 -- input.bin > input.b64
# В одну строку, если это требуется текстовому полю:
base64 -w 0 -- input.bin > input.single-line.b64
```

Восстановление штатной утилитой:

```bash
base64 --decode -- input.b64 > restored-base64.bin
```

Восстановление через Python, если утилита отсутствует:

```bash
python3 decode_file.py base64 input.b64 restored-base64-python.bin
```

Скрипт допускает ASCII-пробелы и переносы, но отклоняет посторонние символы и ошибочное дополнение. Это обычный Base64, не Base64URL: символы `-` и `_` вместо `+` и `/` здесь не поддерживаются. Не удаляйте завершающие `=`. Не применяйте `--ignore-garbage` для сокрытия ошибок повреждённого файла.

## 4. uuencode

Кодирование отдельной утилитой:

```bash
uuencode input.bin restored.bin > input.uue
```

Здесь `input.bin` — источник, `restored.bin` — имя, записанное в заголовок, а `input.uue` — текстовый результат. Заголовок имеет вид `begin 644 restored.bin`; далее идут строки данных, строка нулевой длины и `end`. Число прав может отличаться.

Если `uuencode` не установлен, используйте Python:

```bash
python3 encode_file.py uuencode input.bin input-python.uue
```

Восстановление без установки sharutils:

```bash
python3 decode_file.py uuencode input.uue restored-uue.bin
# Для результата Python-кодировщика:
python3 decode_file.py uuencode input-python.uue restored-uue-python.bin
```

Декодер намеренно восстанавливает файл по пути из аргумента командной строки. Имя и права из заголовка он не применяет. Это позволяет самому выбирать место сохранения. Поддерживается один отдельный классический блок без письма, цитирования и других вложений. Сохраняйте пробелы в конце строк: они могут содержать закодированные данные. Поддерживаются окончания строк LF и CRLF.

**Не используйте `uuencode -m` для этих примеров:** он создаёт другой формат, с Base64-данными и заголовком `begin-base64`, который приведённый uuencode-декодер отклоняет. Модуль Python `uu` не используется, поэтому его удаление в новых Python не мешает работе.

Если sharutils уже установлен, альтернативный способ восстановления:

```bash
uudecode -o restored-uue-tool.bin input.uue
```

## 5. Base32

Кодирование:

```bash
base32 -- input.bin > input.b32
# Без переносов:
base32 -w 0 -- input.bin > input.single-line.b32
```

Восстановление:

```bash
base32 --decode -- input.b32 > restored-base32.bin
# Или без утилиты base32:
python3 decode_file.py base32 input.b32 restored-base32-python.bin
```

Это Base32 по RFC 4648 с алфавитом A–Z, 2–7 и дополнением `=`. Это не Base32hex и не Crockford Base32. Скрипт принимает верхний регистр; не заменяет похожие символы `0/O` и `1/I`. Пробелы и переносы допустимы. Для произвольных больших файлов Base32 менее экономен, чем Base64.

## 6. Ascii85

У Ascii85 нет универсальной штатной команды во всех Linux-дистрибутивах. Чтобы не устанавливать отдельные программы, и кодирование, и декодирование выполняются Python:

```bash
python3 encode_file.py ascii85 input.bin input.a85
python3 decode_file.py ascii85 input.a85 restored-ascii85.bin
```

В этой инструкции используется Adobe-вариант с обрамлением `<~` и `~>`, переносами строк и без сокращения пробелов `y`. Скрипты согласованы именно для такого представления. Это пример обмена отдельными файлами, а не инструмент извлечения потоков из PDF.

Не смешивайте Ascii85 с Git Base85 (`base64.b85encode`) или Z85 (`basenc --z85`): алфавиты и правила различаются. Наличие слова «85» не означает совместимость. Символы Ascii85 могут иметь специальный смысл для shell или разметки, поэтому передавайте результат файлом, а не вставляйте его в команду.

## 7. Скрипт восстановления `decode_file.py`

Один скрипт обслуживает все четыре формата. Он сначала проверяет и декодирует данные, затем создаёт новый выходной файл; существующие файлы не перезаписывает. Ошибки выводятся в stderr с ненулевым кодом завершения.

```python
#!/usr/bin/env python3
import argparse
import base64
import binascii
import re
import sys
from pathlib import Path


def decode_uue(data):
    # Один классический блок begin ... end, без почтовой обёртки.
    lines = data.split(b"\n")
    lines = [line[:-1] if line.endswith(b"\r") else line for line in lines]
    if not lines or not re.fullmatch(rb"begin [0-7]{3,4} .+", lines[0]):
        raise ValueError("Нет заголовка begin MODE NAME; begin-base64 не поддерживается")
    result = bytearray()
    for i, line in enumerate(lines[1:], 1):
        if line in (b"`", b" "):
            if i + 1 >= len(lines) or lines[i + 1] != b"end":
                raise ValueError("После нулевой строки ожидается end")
            if any(line.strip() for line in lines[i + 2:]):
                raise ValueError("Лишние данные после end")
            return bytes(result)
        if not line or not 33 <= line[0] <= 77:
            raise ValueError("Некорректная длина строки uuencode")
        size = line[0] - 32
        expected = 1 + 4 * ((size + 2) // 3)
        if len(line) != expected or any(c < 32 or c > 96 for c in line[1:]):
            raise ValueError("Повреждена строка uuencode; возможно, потеряны пробелы")
        block = binascii.a2b_uu(line)
        if len(block) != size:
            raise ValueError("Неверная длина декодированных данных")
        result.extend(block)
    raise ValueError("Нет нулевой строки и завершающего end")


def main():
    parser = argparse.ArgumentParser(description="Восстановление файла из текстовой кодировки")
    parser.add_argument("encoding", choices=["base64", "base32", "ascii85", "uuencode"])
    parser.add_argument("input", type=Path)
    parser.add_argument("output", type=Path)
    args = parser.parse_args()
    try:
        raw = args.input.read_bytes()
        if args.encoding == "uuencode":
            decoded = decode_uue(raw)
        elif args.encoding == "ascii85":
            # Соглашение: Adobe-обёртка <~ ... ~>, без foldspaces.
            clean = raw.strip(b" \t\r\n\v\f")
            if not (clean.startswith(b"<~") and clean.endswith(b"~>")):
                raise ValueError("Ожидается Ascii85 с маркерами <~ и ~>")
            decoded = base64.a85decode(clean, adobe=True, foldspaces=False)
        else:
            clean = re.sub(rb"[ \t\r\n\v\f]", b"", raw)
            if args.encoding == "base64":
                decoded = base64.b64decode(clean, validate=True)
            else:
                decoded = base64.b32decode(clean, casefold=False)
        # Создание только нового файла: исходник или существующий результат не затирается.
        with args.output.open("xb") as output:
            output.write(decoded)
        print("Восстановлено байт:", len(decoded))
        return 0
    except (OSError, ValueError, binascii.Error) as error:
        print("Ошибка:", error, file=sys.stderr)
        return 1


if __name__ == "__main__":
    sys.exit(main())
```

Скрипт проверяет структуру представления, но не гарантирует целостность содержимого: замена одного допустимого символа другим может остаться незамеченной. Для контроля используйте SHA-256 или побайтное сравнение.

## 8. Скрипт кодирования `encode_file.py`

Нужен для Ascii85, а также позволяет обойтись без `uuencode`, `base64` и `base32`, если Python уже установлен.

```python
#!/usr/bin/env python3
import argparse
import base64
import binascii
import sys
from pathlib import Path

parser = argparse.ArgumentParser(description="Кодирование файла стандартной библиотекой Python")
parser.add_argument("encoding", choices=["base64", "base32", "ascii85", "uuencode"])
parser.add_argument("input", type=Path)
parser.add_argument("output", type=Path)
args = parser.parse_args()
try:
    data = args.input.read_bytes()
    if args.encoding == "base64":
        encoded = base64.b64encode(data) + b"\n"
    elif args.encoding == "base32":
        encoded = base64.b32encode(data) + b"\n"
    elif args.encoding == "ascii85":
        encoded = base64.a85encode(data, adobe=True, wrapcol=80) + b"\n"
    else:
        chunks = [b"begin 644 restored.bin\n"]
        for pos in range(0, len(data), 45):
            chunks.append(binascii.b2a_uu(data[pos:pos + 45]))
        chunks.append(b"`\nend\n")
        encoded = b"".join(chunks)
    with args.output.open("xb") as output:
        output.write(encoded)
except (OSError, ValueError) as error:
    print("Ошибка:", error, file=sys.stderr)
    sys.exit(1)
```

Дополнительные примеры:

```bash
python3 encode_file.py base64 input.bin input-python.b64
python3 encode_file.py base32 input.bin input-python.b32
```

Оба скрипта целиком загружают файл и результат в память. Используйте их, когда размеры уверенно помещаются в RAM с запасом в несколько раз; для многогигабайтных файлов нужна потоковая реализация. Ошибка записи, например заполнение диска, может оставить неполный новый выходной файл. Удалите его или выберите другое имя перед повтором. При ошибке декодирования выходной файл ещё не создаётся.

## 9. Проверка восстановленного файла

Если исходник доступен, сравните байты:

```bash
cmp -- input.bin restored-base64.bin
```

Нет вывода и код завершения `0` — файлы совпадают. Код `1` означает различие, больше `1` — ошибку сравнения.

Или сравните SHA-256:

```bash
sha256sum -- input.bin restored-base64.bin
```

Хеши слева должны быть одинаковыми. Для проверки на другой машине заранее вычислите SHA-256 исходника и сравните его с хешем восстановленного файла. Хеш, полученный по тому же ненадёжному каналу, не подтверждает авторство файла.

Проверка без `sha256sum`:

```bash
python3 - restored-base64.bin <<'PYHASH'
import hashlib
import sys
h = hashlib.sha256()
with open(sys.argv[1], "rb") as file:
    for block in iter(lambda: file.read(1024 * 1024), b""):
        h.update(block)
print(h.hexdigest())
PYHASH
```

Переименовывайте восстановленный файл в исходное расширение при необходимости. Кодирование сохраняет байты, но не исходные время изменения, владельца, ACL и другие метаданные. Для передачи каталога и метаданных сначала создайте архив:

```bash
tar -czf project.tar.gz -- project/
base64 -- project.tar.gz > project.tar.gz.b64
base64 --decode -- project.tar.gz.b64 > restored-project.tar.gz
# Посмотреть состав восстановленного архива:
tar -tzf restored-project.tar.gz
```

## 10. Частые ошибки

| Ошибка | Причина и действие |
| --- | --- |
| `command not found` | Проверьте имя команды; используйте Python-вариант или установите соответствующий пакет |
| `Incorrect padding`, `Invalid base64...` | Возможно, потерян конец файла, выбрана другая разновидность кодировки или добавлен посторонний текст |
| Ошибка uuencode | Проверьте заголовок, `end`, наличие нулевой строки и сохранность завершающих пробелов |
| `File exists` | Python-скрипт не перезаписывает файл; задайте новое выходное имя |
| Файл декодирован, но не открывается | Сравните SHA-256; успешное декодирование само по себе не подтверждает целостность |
| Вставленный текст перестал декодироваться | Передавайте как файл без Markdown-ограждений, почтовых цитат `>` и автоматического форматирования |

## 11. Документация

- [GNU coreutils: Base64](https://www.gnu.org/software/coreutils/manual/html_node/base64-invocation.html)
- [GNU coreutils: Base32](https://www.gnu.org/software/coreutils/manual/html_node/base32-invocation.html)
- [GNU sharutils: uuencode](https://www.gnu.org/software/sharutils/manual/html_node/uuencode-Invocation.html)
- [Python: base64, Base32 и Ascii85](https://docs.python.org/3/library/base64.html)
- [Python: binascii и преобразование uuencode](https://docs.python.org/3/library/binascii.html)
