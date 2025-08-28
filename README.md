## Подготовка

Устанавливаем Zabbix сервер на VM-01
```console
sudo apt update && sudo apt upgrade -y
sudo wget https://repo.zabbix.com/zabbix/7.4/release/ubuntu/pool/main/z/zabbix-release/zabbix-release_latest_7.4+ubuntu22.04_all.deb
sudo dpkg -i zabbix-release_latest_7.4+ubuntu22.04_all.deb
sudo apt install mysql-server -y
sudo systemctl start mysql
sudo apt update && sudo apt upgrade -y
sudo apt install zabbix-server-mysql zabbix-frontend-php zabbix-nginx-conf zabbix-sql-scripts zabbix-agent -y
```
Создаем БД для Zabbix
```console
sudo mysql -uroot -p
mysql> create database zabbix character set utf8mb4 collate utf8mb4_bin;
mysql> create user zabbix@localhost identified by 'zabbix';
mysql> grant all privileges on zabbix.* to zabbix@localhost;
mysql> set global log_bin_trust_function_creators = 1;
mysql> quit;
```
```console
sudo zcat /usr/share/zabbix/sql-scripts/mysql/server.sql.gz | mysql --default-character- set=utf8mb4 -uzabbix -p zabbix
```
```console
sudo mysql -uroot -p
mysql> set global log_bin_trust_function_creators = 0;
mysql> quit;
```
Изменяем конфиг zabbix_server.conf
```console
sudo nano /etc/zabbix/zabbix_server.conf
DBPassword=zabbix
```
Изменяем конфиг nginx.conf
```console
sudo nano /etc/zabbix/nginx.conf
listen 8080;
server_name 10.10.2.73;
```
```console
sudo systemctl restart zabbix-server zabbix-agent nginx php8.1-fpm
sudo systemctl enable zabbix-server zabbix-agent php8.1-fpm
```
Добавляем правила для файрвола
```console
sudo ufw enable
sudo ufw allow 22/tcp
sudo ufw allow 10050/tcp
sudo ufw allow 10051/tcp
sudo ufw allow 80/tcp
sudo ufw allow 8080/tcp
sudo ufw reload
sudo systemctl restart zabbix-server zabbix-agent nginx php8.1-fpm
sudo systemctl restart nginx
```
Русифицируем Zabbix
```console
sudo sed -i "s/# ru_RU.UTF-8 UTF-8/ru_RU.UTF-8 UTF-8/g" /etc/locale.gen
```
Сгенерируем файлы локализации
```console
sudo locale-gen
sudo systemctl restart zabbix-server nginx php8.1-fpm
```
<img width="774" height="253" alt="image" src="https://github.com/user-attachments/assets/c267f6cf-674c-48a6-a7bb-e7cc5a838ac4" />

Создаем нового пользователя в группе Zabbix administrators с ролью Admin role в веб-интерфейсе и даем группе Zabbix administrators доступ до узла Zabbix server

Входим в систему под новым пользователем zabbix_adm zabbix1234

<img width="775" height="271" alt="image" src="https://github.com/user-attachments/assets/2ac65ca6-4540-4c65-a54a-19cab0fb7e7b" />

<br><br>

## Задание №1. Мониторинг сайта yandex.ru

Создаем веб сценарий для узла Zabbix server

<img width="773" height="440" alt="image" src="https://github.com/user-attachments/assets/5c5c0849-ac5a-4901-b28a-53c9b64e179c" />

Шаги

<img width="775" height="111" alt="image" src="https://github.com/user-attachments/assets/c7653820-65c2-49a1-b855-5db930c7b829" />

Включаем следовать перенаправлениям, так как URL редиректит

<img width="773" height="211" alt="image" src="https://github.com/user-attachments/assets/62ba4ad4-5677-411f-b60c-e34ad2907cc8" />

И теги для фильтрации

<img width="773" height="141" alt="image" src="https://github.com/user-attachments/assets/372f0baa-4496-4f8f-ad65-b32c59ba7e03" />

Создаем триггер для недоступности сайта если шаг не выполнен

<img width="772" height="273" alt="image" src="https://github.com/user-attachments/assets/aba45d6e-b6d8-4e8e-99e2-c4dc049d3c77" />

<img width="775" height="240" alt="image" src="https://github.com/user-attachments/assets/e3817d55-8bf6-41f3-b388-05ffd9b8a504" />

Создаем сбой блокируя исходящее подключение на порт 443
```console
sudo ufw deny out 443/tcp
sudo ufw reload
```
<img width="777" height="95" alt="image" src="https://github.com/user-attachments/assets/9e54de12-9118-42bf-8523-855e65f90f7a" />

Восстанавливаем соединение удаляя правило для 443 порта
```console
sudo ufw status numbered
sudo ufw delete 6
sudo ufw delete 11
sudo ufw reload
```
Алерт закрылся

<img width="773" height="105" alt="image" src="https://github.com/user-attachments/assets/5808d547-f3ac-4863-9584-1b449fae20d9" />

<br><br>

## Задание №2. Проверка доступности сервера yandex.ru

Создаем для узла сети Zabbix server элемент данных

<img width="773" height="209" alt="image" src="https://github.com/user-attachments/assets/ee578912-5b5a-4780-99a6-125f63cd1451" />

Устанавливаем пакет fping
```console
sudo apt update && sudo apt upgrade -y
sudo apt install fping -y
```
Исправляем путь до fping в zabbix_server.conf
```console
sudo nano /etc/zabbix/zabbix_server.conf
FpingLocation=/usr/bin/fping
sudo systemctl restart zabbix-server zabbix-agent nginx php8.1-fpm
```

Создаем триггер если пинг не проходит

<img width="773" height="289" alt="image" src="https://github.com/user-attachments/assets/db7d2dfe-2bb2-4e31-b65d-88cb8e50f332" />

<img width="773" height="271" alt="image" src="https://github.com/user-attachments/assets/9e01cd62-5cbf-4535-99b7-439d8ea4e3c9" />

Имитируем сбой
```console
sudo nano /etc/hosts
172.0.0.1 yandex.ru
```

Сработали оба триггера

<img width="773" height="138" alt="image" src="https://github.com/user-attachments/assets/279a29cc-c75f-429c-89f4-a3f9b03327c8" />

Комментируем добавленную строку в /etc/hosts

<img width="772" height="126" alt="image" src="https://github.com/user-attachments/assets/f989b5e8-5895-4ef2-a69d-5f34dd2ed5df" />

<br><br>

## Задание №3. Установка зависимостей проверок

Настраиваем зависимость триггера «Сайт yandex.ru недоступен» от триггера «Сервер Яндекс недоступен» чтобы при недоступности сервера появлялась только одна проблема

<img width="773" height="152" alt="image" src="https://github.com/user-attachments/assets/da2d1bf0-d9f0-4a12-8058-f1cbb24939b2" />

Проверяем сбой для сервера и восстанавливаем связь

<img width="774" height="150" alt="image" src="https://github.com/user-attachments/assets/3b2661d3-1b75-458c-8b6a-23467d1ab792" />

Проверяем сбой для сайта и восстанавливаем связь

<img width="775" height="175" alt="image" src="https://github.com/user-attachments/assets/e6e1ba79-bba9-4430-a07e-53572e5ac30c" />


