---
title: СПО. ЛР № 2. Указания к выполнению
lang: ru

---

Разделы, отмеченные `[C++]`, могут представлять интерес для всех вариантов.

---

Почти все процессы так или иначе взаимодействуют с другими:

* При работе из командной строки ввод в оболочку (`cmd.exe` в Windows)
    передается вызываемой программе, а вывод забирается у нее
    и отображается в оболочке; используются анонимные каналы.
* Весь интернет — это среда для обмена данными между процессами-клиентами
    (например, браузерами) и программами-серверами (соответственно,
    web-серверами).  Процессы при этом запущены на разных машинах.
* Программы, работающие с сервером баз данных, передают ему запросы
    и получают ответы.  Это может происходить как на одной машине,
    так и по сети.  Например, MS SQL Server позволяет общаться с ним
    по сети, через именованные каналы или через разделяемую память.
* Обычные программы получают команды и запросы системным службам,
    в Windows обычно через механизм RPC (remote procedure call)
    или через оконные сообщения;
    в \*nix — обычно через локальные сокеты (UNIX domain sockets).

Не взаимодействуют с другими только самые простые задачи (хотя иногда
очень важные, например, служебные потоки ядра ОС).

В лабораторной работе изучаются именованные каналы, анонимные каналы
и разделяемая память (как механизмы, присутствующие во всех популярных ОС)
и Windows mailslots (как реализующие популярную во многих ОС концепцию).
Не изучаются сеетвые сокеты (как обширная тема, выходящая за пределы
дисциплины) и RPC (как специфичный для Windows механизм).


# Вариант 1.  Обмен сообщениями через разделяемую память

Механизм разделяемой памяти (shared memory) позволяет создать в каждом
из взаимодействующих процессов область виртуальной памяти, содержимое которой
будет общим для всех участвующих процессов.  Любой из этих процессов может
записать данные в разделяемую память, и другие процессы смогут их считать.
Это самый быстрый способ IPC, потому что работа с памятью не требует
системных вызовов (кроме, возможно, страничных прерываний).
Разделяемая память работает только в пределах машины.

В Windows разделяемая память реализована на основе механизма проецирования
файлов в память (file mapping).  Проецирование файлов в память позволяет создать
область памяти, содержимое которой будет совпадать с файлом или его частью.
Чтение из этой области будет работать как чтение из файла, запись в нее —
как запись в файл.  Несколько процессов могут проецировать в свою память
одни и те же файлы или их части, тогда содержимое этих областей памяти будет
разделяемым между ними.  Поэтому работа с разделяемой памятью начинается
с функции [`CreateFileMapping()`][win32/CreateFileMapping].  Ее параметры:

* `HANDLE hFile` — дескриптор открытого файла, который проецируется.
    Если вместо него передать `INVALID_HANDLE_VALUE`, будет использована
    часть системного файла подкачки.

* `LPSECURITY_ATTRIBUTES lpFileMappingAttributes` — указатель на атрибуты
    безопасности разделямой памяти (ограничения доступа к ней).
    Если не нужно особых условий, допускается передать `NULL`.

`DWORD flProtect` — режим доступа к области памяти: только для чтения,
    для чтения и записи (`PAGE_READWRITE`) и другие варианты.

* `DWORD dwMaximumSizeHigh, DWORD dwMaximumSizeLow` — максимально допустимый
    размер области, которую смогут спроецировать процессы при помощи
    создаваемого отображения файла в память.  Размер — 64-битное значение,
    поэтому передается в двух 32-битных переменных: для старших и младших
    разрядов.

`LPCSTR lpName` — имя файлового отображения, по которому другие процессы
    могут отыскать его функцией [`OpenFileMapping()`][win32/OpenFileMapping]

`CreateFileMapping()` возвращает дескриптор файлового отображения.

Далее этот дескриптор передается функции
[`MapViewOfFile()`][win32/MapViewOfFile], которая проецирует часть
файлового отображения в адресное пространство процесса,
возвращая указатель на область памяти с проекцией:
```
LPVOID memory = MapViewOfFile(...);
```

Также `MapViewOfFile()` передаются желаемые права доступа к участку памяти
(например, `FILE_MAP_ALL_ACCESS` для полного доступа), смещение от начала
файлового отображения (тоже в виде двух чисел, в случае ЛР это нули)
и размер проецируемой области (имеет смысл проецировать все отображение).

---

## [C++] Нетипизированные указатели, приведение типов и `auto`

`LPVOID` (псевдоним для `void*`) — так называемый *нетипизированный указатель:*
адрес памяти, по которому могут находиться данные любого типа.  От обычных
указателей он отличается тем, что его нельзя разыменовать (компилятору
неизвестно, какого типа значение должно получиться после разыменования).
Поэтому для его использования нужно *приведение типов (type cast):*
```
char* message = reinterpret_cast<char*>(memory);
```

Это означает: взять адрес, хранящийся в `memory`, рассмотреть его как адрес
символов, и поместить тот же адрес в указатель на символы `message`.
Вместо конструкции `reinterpret_cast` можно встретить короткую форму в стиле C:
```
char* message = (char*)memory;
```

Однако громоздкая форма введена в C++ намеренно: приведение типов — опасная
операция в том смысле, что компилятор не может проверить корректность
приведения типов (программист утверждает, что по адресу `memory` будет строка,
но компилятор не может проверить, не ошибается ли программист).  Поэтому чем
меньше в программе приведений типов и чем более они заметны, тем лучше.

Чтобы не писать `char*` слева и справа (очевидно, что раз `memory` приводится
к типу `char*`, тип `message` будет таким же) можно использовать ключевое
слово `auto`:
```
auto message = reinterpret_cast<char*>(memory);
```

---

Так можно записать данные в разделяемую область:
```
char input[128];
fgets(input, sizeof(input), stdin);
strcpy(message, input);
```

Функция `fgets()` из стандартной библиотеки C++ `<cstdio>` считывает строку
из файла.  В данном случае в качестве файла выступает стандартный поток
ввода `stdin`.  Также функция принимает буфер, куда сохраняется строка,
и его размер.

Функция `strcpy()` копирует данные из второй строки C в первую.
(Напомним: строка C — указатель на массив символов; ее копирование —
копирование каждого символа из одной области памяти в другую.)
В данном случае сообщение из буфера `input` копируется в разделяемую память
по адресу, находящемуся в `message`.

Для считывания данных не нужно и промежуточного буфера:
```
printf("Message from shared memory: %s\n", message);
```

По окончании работы с разделяемой памятью нужно прекратить ее проецирование
функцией [`UnmapViewOfFile()`][win32/UnmapViewOfFile].

Также нужно освободить объект-отображение.  Для этого используется
универсальная функция для закрытия дескрипторов
[`CloseHandle()`][win32/CloseHandle].

[win32/CreateFileMapping]: https://docs.microsoft.com/en-us/windows/desktop/api/WinBase/nf-winbase-createfilemappinga
[win32/OpenFileMapping]: https://docs.microsoft.com/en-us/windows/desktop/api/winbase/nf-winbase-openfilemappinga
[win32/MapViewOfFile]: https://msdn.microsoft.com/en-us/df9f54cd-b2de-4107-a1c5-d5a07045851e
[win32/UnmapViewOfFile]: https://msdn.microsoft.com/en-us/aa366882
[win32/CloseHandle]: https://msdn.microsoft.com/en-us/library/ms724211(v=VS.85).aspx


# Вариант 2.  Хранилище значений по ключу с доступом через именованные каналы

Именованные каналы (named pipes) позволяют ровно двум процессам передавать
последовательность сообщений или просто байтов.  С использованием пары каналов
(в \*nix) или одного дуплексного канала (в Windows) это удобно для сценариев,
когда процессы делают запросы и получают на них ответы.  Каналы уступают
в скорости разделяемой памяти (каждое обращение к каналу — системный вызов),
но не требуют обращений к оборудованию\*, данные передаются через память.
Еще одно удобство именованных каналов в том, что работа с ними ведется
как с файлами, например, можно процессу, ожидающему файл с исходными данными
передать канал, данные в котором генерируются другим процессом.

**\*** Именованные каналы позволяют сообщаться и процессам на различных
машинах, в этом случае, очевидно, используется оборудование — сетевая карта.

## Сервер

Для ввода данных в этом варианте целесообразно использовать средства C++
из заголовочного файла `<iostream>` и строки C++ из `<string>`.
Например, так вводится имя канала:
```
std::string name;
std::cin >> name;

auto path = "\\\\.\\pipe\\" + path;

auto pipe = CreateNamedPipe(path.c_str(), ...);
```

Полный путь к каналу формируется из переменной `name` в переменной `path`.
Сравнивая его с документацией, можно видеть, что в строках специальный символ
`\ ` требует удвоения.  (Внимание: программы-примеры требуют вводить полный
путь.)

Выражение `path.c_str()` получает из объекта-строки `path` указатель
на строку C с ее символами путем вызова метода `c_str()`.

Функция [`CreateNamedPipe()`][win32/CreateNamedPipe], помимо имени канала,
принимает также режим открытия (принимающий, передающий или дуплексный)
и режим работы канала (ориентированный на байты или на сообщения).
Размеры буферов приема и передачи целесообразно указать порядка типового
размера сообщения, например, по 64.  Таймаут операций и атрибуты безопасности
для данный ЛР не важны, можно указать их как `0` и `NULL` соответственно.
`CreateNamedPipe()` возвращает дескриптор канала.

Этот дескриптор принимает [`ConnectNamedPipe()`][win32/ConnectNamedPipe].
Ее второй параметр предназначен для асинхронной работы, выходящей за рамки ЛР,
поэтому его можно сделать `NULL`.

Функция [`ReadFile()`][win32/ReadFile] принимает дескриптор канала и буфер
для считываемых из него данных с размером этого буфера.  В качестве буфера
можно использовать строку C++ следующим образом:
```
std::string command(64, '\0');
ReadFile(pipe, &command[0], command.size(), NULL, NULL);
```

Первая инструкция создает переменную `command` типа `std::string`, состоящую
из 64 символов конца строки (`'\0'`).  Вторая инструкция передает указатель
`&command[0]` на символы в строке `command` как буфер для `ReadFile()`.
Размер буфера получается вызовом метода `size()` у `command`.

---

### [C++] Разбор строк

Вычитывать отдельные слова из строки-команды можно с помощью средств
стандартного заголовочного файла `<sstream>`:
```
std::istringstream parser{command};
std::string keyword;
parser >> keyword;
```

Переменная `parser` типа `std::istringstream` — это *поток ввода* из строки,
то есть `parser` работает так же, как `std::cin`, но читает не с клавиатуры,
а из строки `command`.  В данном случае на последней строки считывается
первое слово в переменную `keyword`.  Если повторить эту операцию,
будет прочитано второе слово.  Например, так можно обрабатывать команду `set`:
```
if (keyword == "set") {
    std::string name;
    std::string value;
    parser >> name >> value;
```

---

Как сохранить в словаре значение `value` по ключу `name`?  Работа со словарями
типа `std::map` описана в [пособии][maps-and-streams].  Переменную-словарь
нужно завести в начале функции `main()`.  Для продолжения обработки
команды `set` нужно записать значение в словарь (названный `data`):
```
    data[name] = value;
```

Ответ сервера записывается в канал функцией [`WriteFile()`][win32/WriteFile],
которая аналогична `ReadFile()`:
```
    std::string response = "acknowledged";
    WriteFile(pipe, response.c_str(), response.size(), NULL, NULL);
}
```

Структура программы-сервера следующая: цикл работы с клиентами (одна итерация
на одно подключение, выход по запросу пользователя сервера), внутри которого
цикл обработки команд клиента (выход по команде `quit`).  Таким образом,
на верхнем уровне сервер устроен так:
```
pipe = CreateNamedPipe(...)

while (true) {
    ConnectNamedPipe(pipe, ...);

    while (true) {
        ReadFile(pipe, ...);

        // обработка различных команд (опущена)

        if (command == "quit") {
            DisconnectNamedPipe(pipe);
            break;
        }
    }

    // запрос продолжения у пользователя, break при отказе (опущено)
}
```


## Клиент

Подключение к именованному каналу выполняется универсальной функцией
[`CreateFile()`][win32/CreateFile].  Ключевой параметр — *полное* имя канала.
Желаемые режимы доступа `GENERIC_READ | GENERIC_WRITE`, так как клиент
и пишет в канал команды, и читает ответы на них.  Разделения доступа и особых
атрибутов безопасности не требуется, можно передать `0` и `NULL`.
Подключение производится к существующему каналу (режим `OPEN_EXISTING`),
особых атрибутов у канала нет (`FILE_ATTRIBUTE_NORMAL`).

[win32/CreateNamedPipe]: https://docs.microsoft.com/en-us/windows/desktop/api/Winbase/nf-winbase-createnamedpipea
[win32/ConnectNamedPipe]: https://msdn.microsoft.com/en-us/library/Aa365146(v=VS.85).aspx
[win32/CreateFile]: https://docs.microsoft.com/en-us/windows/desktop/api/fileapi/nf-fileapi-createfilea
[win32/ReadFile]: https://docs.microsoft.com/en-us/windows/desktop/api/fileapi/nf-fileapi-readfile
[win32/WriteFile]: https://docs.microsoft.com/en-us/windows/desktop/api/fileapi/nf-fileapi-writefile
[maps-and-streams]: https://docs.google.com/document/d/12HaYKjOMaqygGyenepie4jfWPTkzqsm-ZYqFnSuT8PQ


# Вариант 3. Обмен сообщениями через механизм почтовых ящиков (mailslots)

Механизм почтовых ящиков (mailslots) изначально реализован в Windows
для совместимости с OS/2.  Поскольку развитие OS/2 прекратилось в 1990-е,
а в других ОС нет mailslots, это уникальный для Windows и непопулярный
в целом механизм.  В \*nix аналогичная задача — отправка сообщений от многих
клиентов одновременно одному серверу — решается обычно через локальные сокеты
(UNIX domain sockets), отсутствующие в Windows.  Поэтому, хотя конкретно
mailslots и мало распространены, реализуемый ими принцип весьма практичен.

Функцию [`CreateMailslot()`][win32/CreateMailslot] требуется вызывать
с указанием особых атрибутов безопасности, как описано в задании.
С учетом приведенного там же кода это делается так:
```
auto attributes = create_security_attributes();
auto mailslot = CreateMailslot(..., &attributes);
```

Об использовании `ReadFile()` и `WriteFile()` см. указания к варианту 2.

[win32/CreateMailslot]: https://docs.microsoft.com/en-us/windows/desktop/api/Winbase/nf-winbase-createmailslota


# Вариант 4. Запуск дочернего процесса с перенаправлением потоков ввода-вывода

Перед выполнением рекомендуется внимательно ознакомиться с заданием
и пояснениями к нему.

Укрупненный алгоритм работы программы:

1. Создать анонимный канал для ввода и для вывода (по два дескриптора).
2. Сделать дескрипторы тех концов каналов, которые будут использоваться
    дочерним процессом, наследуемыми.
3. Создать дочерний процесс, передав ему дескрипторы-концы каналов
    для использования в качестве стандартного ввода и вывода.
4. В цикле считывать команду с клавиатуры, записывать ее в канал ввода,
    читать результат из канала вывода и печатать на экран.

Чтобы дескрипторы каналов могли быть наследуемыми, нужно при создании
канала передавать атрибуты безопасности:
```
SECURITY_ATTRIBUTES attributes;
attributes.nLength = sizeof(SECURITY_ATTRIBUTES);
attributes.bInheritHandle = TRUE;
attributes.lpSecurityDescriptor = NULL;
```

Анонимные каналы создаются функцией [`CreatePipe()`][win32/CreatePipe],
например, для канала ввода:
```
HANDLE input_pipe_read_end;
HANDLE input_pipe_write_end;
CreatePipe(&input_pipe_read_end, &input_pipe_write_end, &attributes, 0);
```

Здесь `input_pipe_read_end` — конец канала, из которого производится чтение,
то есть тот конец, который будет передан дочернему процессу.

Канал вывода создается аналогично для второй пары дескрипторов,
только из них в дочерний процесс будет передаваться конец для записи в канал.

В дополнение к атрибутам безопасности необходимо разрешить наследование
функцией [`SetHandleInformation()`][win32/SetHandleInformation]
для тех дескрипторов-концов, которые остаются в текущем процессе
(на примере канала ввода, для канала вывода аналогично):
```
SetHandleInformation(input_pipe_write_end, HANDLE_FLAG_INHERIT, 0);
```

Для создания дочернего процесса нужно подготовить структуру
[`STRATUPINFO`][win32/STARTUPINFO], указав в ней концы каналов
для передачи в создаваемый процесс и необходимые флаги:
```
STARTUPINFO startup_info;
ZeroMemory(&startup_info, sizeof(startup_info));
startup_info.cb = sizeof(startup_info);
startup_info.hStdInput = input_pipe_read_end;
startup_info.hStdOutput = output_pipe_write_end;
startup_info.hStdError = output_pipe_write_end;
startup_info.dwFlags |= STARTF_USESTDHANDLES;
```

Функция [`ZeroMemory()`][win32/ZeroMemory] заполняет нулями участок памяти,
заданный указателем на начало и длиной.

Знак `|=` — оператор сокращенного присваивания, совмещенного
с побитовым «ИЛИ», то есть с добавлением флага в комбинацию:
```
startup_info.dwFlags = startup_info.dwFlags | STARTF_USESTDHANDLES;
```

При запуске дочернего процесса функцией [`CreateProcess()`][win32/CreateProcess]
для большей части параметров годится значение по умолчанию, то есть передается
`0` или `NULL`.
```
PROCESS_INFORMATION pi;
CreateProcess(
    NULL,
    "cmd.exe",
    NULL,
    NULL,
    TRUE,
    0,
    NULL,
    NULL,
    &startup_info,
    &pi);
```

Необходимо указать лишь имя исполняемого файла (`"cmd.exe"`),
признак необходимости наследовать дескрипторы (`TRUE`),
а также указатели на `STARTUPINFO` и `PROCESS_INFORMATION`
с результатами вызова (дескрипторами процесса и потока).

Далее следует цикл отображения приглашений командной строки,
чтения команд и выдачи результатов.

Чтение приглашения выполняется функцией `ReadFile()`, о которой написано
в указаниях к варианту 2.  Следует обратить внимание, что дочерний процесс
не пишет в конце своего вывода символ `'\0'`, поэтому нужно выводить ровно
те символы, которые были получены, а не просто весь буфер, так как нельзя
рассчитывать на наличие у него `'\0'` в конце:
```
DWORD bytes_read;
do {
    char buffer[64];
    ReadFile(output_pipe_write_end, buffer, sizeof(buffer), &bytes_read, NULL);
    fwrite(buffer, bytes_read, 1, stdout);
```

Здесь стандартная функция С++ для записи в файл `fwrite()` (из `<cstdio>`)
получает в качестве файла стандартный поток вывода `stdout`.

Заканчивать чтение очередного блока вывода дочернего процесса нужно тогда,
когда будет прочитано 0 байтов:
```
} while (bytes_read != 0);
```

Напомним, цикл `do { ... } while (...);` в C++ аналогичен циклу
`repeat... until` в Pascal, но с инвертированным условием.

Запрос у пользователя команды вкупе с проверкой вежливости можно выполнить так:
```
const char PLEASE[] = "please";

char* input = NULL;
char buffer[256];
while (!input) {
    fgets(buffer, sizeof(buffer), stdin);
    if (!strncmp(buffer, "thanks", 6)) {
        break;
    }
    else if (strncmp(buffer, "please", 6)) {
        fprintf(stderr, "Please ask politely!\n> ");
    }
    else {
        input = buffer + sizeof(PLEASE);
    }
}
```

Все функции здесь — из стандартной библиотеки C++.  Разобрать его или написать
аналог самостоятельно предлагается в качестве упражнения.  По окончании цикла
либо будет `input == NULL` (и тогда нужно выйти из программы), либо `input`
будет указывать на начало непосредственно команды (без слова `please`).

Команда передается дочернему процессу путем записи в канал ввода
через дескриптор `input_pipe_write_end` функцией `WriteFile()`
(см. указания к варианту 2).  В качестве буфера ей передается `input`,
его длину можно вычислить стандартной функцией `strlen()`.

[win32/CreatePipe]: https://msdn.microsoft.com/en-us/library/Aa365152(v=VS.85).aspx
[win32/CreateProcess]: https://docs.microsoft.com/en-us/windows/desktop/api/processthreadsapi/nf-processthreadsapi-createprocessa
[win32/SetHandleInformation]: https://msdn.microsoft.com/en-us/library/windows/desktop/ms724935(v=vs.85).aspx
[win32/STARTUPINFO]: https://docs.microsoft.com/en-us/windows/desktop/api/processthreadsapi/ns-processthreadsapi-_startupinfoa
[win32/ZeroMemory]: https://msdn.microsoft.com/en-us/library/windows/desktop/aa366920(v=vs.85).aspx