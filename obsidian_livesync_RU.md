### Требования

- Docker - для разворачивания необходимых контейнеров
- CouchDB - здесь будут хранится все файлы обсидиан и их версии, livesync работает с этой БД
- Nginx - для проксирования запросов на сервере в CouchDB и установки сертификатов
- DuckDNS - бесплатно получаем dns domain, к сожалению, в плагине livesync нельзя использовать ip сервера, так как obsidian запрещает использовать self-signed сертификаты, а проверенные сертификаты мы можем установить только на доменное имя. Если у вас есть уже купленный домен, то можно не использовать duckDNS или аналоги с беплатными доменами
- Let's Encrypt - генериуем бесплатный 90 дневный проверенный сертификат, работает в РФ

### Установка

Мы предполагаем, что мы поднимаем весь сервис с нуля и создаем контейнеры только для livesync.

#### Регистрируем домен duckdns

1) Переходим на https://www.duckdns.org/
2) Логинимся
3) Добавляем домен, указываем IP своего хоста и получаем адрес, например http://scooperd9i.duckdns.org

#### Генериуем сертификат Let's Encrypt

1) Генерируем сертификат с помощью certbot
```bash
sudo apt install certbot python3-certbot-nginx
sudo certbot certonly --standalone -d scooperd9i.duckdns.org
```
2) Достаем пути, где сохранились сертификаты
```bash
certbot certificates
```
Команда выведет пути, где хранится сертификаты. Будьте внимательны, что там скорее всего будут хранится семилинки, нам нужны реальные файлы
#### Создаем и запускаем docker-compose файл для Nginx

1) Создаем папки
```bash
mkdir ~/nginx-livesync
cd ~/nginx-livesync
mkdir proxy
```

2) Создаем файл конфигурации в папке proxy, `nano proxy/nginx.conf`
```nginx
server {
  listen 443 ssl;
  server_name scooperd9i.duckdns.org;
  client_max_body_size 100M;
  ssl_certificate /etc/nginx/ssl/fullchain.pem; #указываем путь до сертификатов от Let's Encrypt
  ssl_certificate_key /etc/nginx/ssl/privkey.pem;
	 location / {
	    proxy_pass http://couchdb:5984;
	    proxy_redirect off;
	    proxy_buffering off;
	    proxy_set_header Host $host;
	    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
	 }
	 
	 location /couchdb {
	    rewrite ^ $request_uri;
	    rewrite ^/couchdb/(.*) /$1 break;
	    proxy_pass http://couchdb:5984$uri;
	    proxy_redirect off;
	    proxy_buffering off;
	    proxy_set_header Host $host;
	    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
	}

	location /_session {
	    proxy_pass http://couchdb:5984/_session;
	    proxy_redirect off;
	    proxy_buffering off;
	    proxy_set_header Host $host;
	    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
	}
}
```

3) Создаем docker-compose и запускаем
```yaml
services:
  proxy:
    image: nginx
    networks:
      - nginx
    volumes:
      - type: bind
        source: ./proxy/nginx.conf
        target: /etc/nginx/conf.d/default.conf
        read_only: true
      - ./certs:/etc/nginx/ssl
    ports:
      - 443:443

networks:
  nginx:
    driver: bridge
```

4) Проверяем, по адресу https://scooperd9i.duckdns.org должна открываться страница приветствия Nginx
#### Создаем и запускаем docker-compose файл для CouchDB

1) Создаем папки
```bash
mkdir ~/couchdb-livesync
cd ~/couchdb-livesync
```

2) Создаем docker-compose и запускаем
```yaml
version: '3.8'

services:
  couchdb:
    image: couchdb:latest
    container_name: couchdb
    restart: unless-stopped
    networks:
      - nginx_nginx
    environment:
      - COUCHDB_USER=admin
      - COUCHDB_PASSWORD=changeit #заменяем пароль на свой
    volumes:
      - ./couchdb_data:/opt/couchdb/data

networks:
  nginx_nginx:
    external: true
```

3) Здесь необходимо настроить CouchDB из репозитория livesync, запускаем скрипт из репо, подставляем данные из учетки CouchDB, указанные в docker-compose
```bash
curl -s https://raw.githubusercontent.com/vrtmrz/obsidian-livesync/main/utils/couchdb/couchdb-init.sh | hostname=https://scooperd9i.duckdns.org username=admin password=changeit bash
```

4) Должны получить такой ответ после запуска скрипта:
```
-- Configuring CouchDB by REST APIs... -->
{"ok":true}
""
""
""
""
""
""
""
""
""
<-- Configuring CouchDB by REST APIs Done!
```

5) Все должно работать, чтобы убедиться, можем зайти в админку CouchDB
   https://scooperd9i.duckdns.org/_utils/
   
#### Настраиваем клиент (базово)
1) Запускаем Obsidian
2) Переходим в Community Plugins, включаем
3) Ищем плагин `Self-hosted LiveSync` и включаем его
4) Плагин предложить импорировать Settings Url, откажемся и введем все настройки в ручную
5) Переходим в настройки плагина и заходим во вкладку `Remote Configuration`
6) Настраиваем следующим образом
```
[Remote Server]
Remote Type: CouchDB

[CouchDB]
Server URI: https://scooperd9i.duckdns.org
Username: admin
Password: changeit
Database Name: test_name #тут указываете как хотите, я лично указываю название БД по названию хранилища
```
7) Нажимаем Test и Check и проверяем, что все хорошо
8) Донастраиваем шифрование данных в этой же вкладке
```
[Privacy & Encryption]
End-to-End Encryption: yes
Passphrase: someseedpassword #тут придумываем ключ для шифрования, запоминаем и сохарняем его где-нибудь
```
9) Нажимаем Apply и синхронизация работает
10) Тут плагин может предлагать донастроить разные фичи, это уже на усмотрение пользователя

#### Подключаем клиенты
##### На стороне источника настроек
Чтобы было просто подключать другие устройства, заходим в настройки плагина и во вкладку Setup
Нажимаем Copy the current settings to a Setup URI, вводим новый пароль для шифрования настроек и плагин копирует его в буфер обмена, делимся этим урлом с устройствами, которые хотим подключить

##### На стороне клиента
Устанавливаем плагин `Self-hosted LiveSync` и здесь уже соглашаемся импортировать настройки из Setup URI, вставляем сюда URI из предыдущего шага, указываем пароль для дешифрования и наши настройки заимпортировались, теперь данный девайс будет синхронизироваться тоже с сервером

#### Добавляем новые хранилища
Для добавления новых хранилищ, достаточно также сделать шаги начиная с `Настраиваем клиент (базово)`, только в поле Database name указываем другое значение, например название хранилища, имена должны быть уникальными
