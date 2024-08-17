## Лекция 7. Веб-приложение в контейнерах.
### 7.1 Фронтенд Nginx.
Добавим фронтенд к нашему сайту. Это можно сделать 2 способами.
    
  ![Один сервер](https://github.com/asbabushkin/Docker/blob/Lesson7/Lesson7/one-server%20scheme.png)

  При использовании одного сервера на бэкенде расположены папки с шаблонами html-страниц и static-файлами. Получив запрос, бэкенд получает данные из БД и заполняет ими шаблоны html-страниц, используя шаблонизатор (например, Jinja, Django Template Language). Полученную html-страницу отобразит пользователю.  
  Более совершенный вариант - выделить под фронтенд (статику) отдельный сервер:  
    ![Два сервера](https://github.com/asbabushkin/Docker/blob/Lesson7/Lesson7/two-server%20scheme.png)
В этом случае запрос от пользователя принимает фронт-сервер и возвращает назад html-страницу, которая содержит запрос на определенную ручку бэк-сервера. Бэк-сервер обрабатывает запрос (обращается к бд) и возвращает json-файл с данными, которые используются для заполнения html-страницы.  
Вариант с одним сервером хорош для несложных приложений с небольшой нагрузкой.  
Мы же будем использовать второй вариант.
Итак, в нашем первом контейнере будет Nginx, во втором - бэкенд, а в третьем база данных Postgres.
Nginx - это http-сервер (обработчик http-запросов). Его задачи: 
* хранение и раздача статических файлов (html, css, js);
* проксирование запросов на бэкенд.
![Nginx scheme](https://github.com/asbabushkin/Docker/blob/Lesson7/Lesson7/nginx%20scheme.png)
Описание схемы: пользователь вбивает адрес сайта и попадает на сервер с Nginx, бэкендом и бд. По номеру порта сервер понимает, что запрос следует направить Nginx'у. Nginx возвращает браузеру файл index.html с зашитыми в него ссылками на файлы стилей и js. Браузер снова отправляет запрос на сервер, тот переадресовывает его в Nginx, который отдает браузеру файлы style.css и script.js-файлы. Файлы script.js содержит запрос на получение данных из бд, поэтому вновь посылает запрос на сервер, который вновь отдает его Nginx. Но тот понимает, что запрос предназначен не ему и пропускает запрос, который попадает во второй сервер с бэкендом. Бэкенд по запросу получает данные из бд и по цепочке передает их назад.  
Файл конфигурации nginx представлен ниже:  
```
user www-data;
worker_processes 2;
events {
    worker_connections 2048;
}
http {
    map $uri $base {
        ~/(?<file>[^/]*)$ $file;
    }
    server {
        listen 80;
        server_name _;

        location /api {
            proxy_pass http://backend:8000/api;
        }
        location / {
            root /nginx/static;
            try_files /$base /index.html =404;
        }
    }

    include /etc/nginx/sites-enabled/*.conf;
    include mime.types;
}
```
* listen 80 - порт, который слушает nginx;
* server_name - адрес сервера;
* location /api {proxy_pass http://backend:8000/api;} - проксирование (проброс) запроса, приходящего на /api на бэк-сервер (backend - имя контейнера с бэком);
* location / {root /nginx/static; try_files /$base /index.html =404;} - при запросе на просто слэш nginx вернет статику.

### 7.2 Multi-stage build
Сейчас наш фронтенд состоит из множества компонентов, к тому же написан на Typescript. Из всего этого нужно собрать 3 файла: css, js, index.html и положить их в контейнер с nginx. Это можно сделать локально и положить в образ уже готовые файлы, но при таком подходе нам придется пересобирать их заново каждый раз при обновлении образа. Лучше использовать многоэтапную сборку (multi-stage build).
Докер-файл многоэтапной сборки имеет несколько инструкций FROM и результаты первой сборки можно использовать во второй. Это позволяет значительно сократить размер итогового образа, т.к. в него не включаются исходники для создания первого.  

### 7.2 Сборка фронтенда
Создадим докер-файл для фронтенда:
```
FROM node:17 AS BUILD
WORKDIR /app
COPY ./todo-list/package.json ./package.json 
RUN npm i # устанавливка зависимостей и из package.json

COPY ./todo-list ./
RUN npm run build # запуск первой сборки
# собраны артефакты: index.html, css, js которые нужно передать в NGINX

FROM nginx # Второй этап сборки

COPY --from=BUILD /app/dist/index.html /nginx/static/index.html # копируем собранный index.html
COPY --from=BUILD /app/dist/static/css /nginx/static/ # копируем собранный css
COPY --from=BUILD /app/dist/static/js /nginx/static/ # копируем собранный js
```
Докер-файл для бэкенда:
```
FROM python:3.8
COPY requirements.txt requirements.txt
RUN python -m pip install --upgrade pip && pip install -r requirements.txt
WORKDIR /app
COPY wsgi.py wsgi.py
COPY app.py app.py
EXPOSE 8000
ENTRYPOINT [ "gunicorn", "--bind", "0.0.0.0:8000", "wsgi:app" ] # Поднимем вебсервер
```
Докер-файл для бд:
```
FROM postgres:14
COPY ./init_db.sh /docker-entrypoint-initdb.d/init_db.sh # копируем скрипт, заполняющий бд
```
содержит скрипт, заполняющий бд:  
```
#!/bin/bash
set -e
psql -v ON_ERROR_STOP=1 --username "$POSTGRES_USER" --dbname "$POSTGRES_DB" <<-EOSQL
  CREATE TABLE IF NOT EXISTS app_table
  (
    id text NOT NULL,
    text text NOT NULL,
    status text NOT NULL
  )
EOSQL
```
После сборки образа фронтенда видно, что образ первой сборки весит 1,35 ГБ, а итоговый - лишь 142 МБ.

