## Лекция 9. Веб-приложение в docker-compose.
### 9.1. Инструкции docker-compose.
Создадим docker-compose.yaml файл для поднятия наших сервисов. Он будет иметь следующие ключи:  
```
version: # версия
services: # контейнеры
volumes: # вольюмы
networks: #  сети
```

#### Volumes
Имя вольюма можно не указывать, тогда оно будет сформировано автоматически (имя папки + имя бд).
```
volumes:
    todo_db:
```
А можно задать явно:
```
volumes:
    todo_db:
        name: todo_db
```
#### Networks
Имена сетей задаются аналогично именам вольюмов.

#### Services
В ключах блока сервисов указываем имена наших контейнеров:  
```
services:
    db:

    backend:

    nginx:

```
Основные инструкции для описания контейнеров нам уже знакомы:
  ![Container commands](https://github.com/asbabushkin/Docker/blob/Lesson9/Lesson9/Container%20commands.png)

Но в docker-compose есть и новые важные опции:  
* restart и restart_policy - установить возможность и условия перезапуска контейнера. Например, при падении перезапускаться всегда.
* replicas - количество копий контейнера, которые нужно поднять
* depends_on - выставляет зависимость одного контейнера от другого. В нашем приложении мы запускаем сначала бд, затем - бэк, после - фронт. Значит, для контейнера с бэком условием будет являться наличие контейнера с бд.
* healthcheck - запустить некую проверку после поднятия контейнера. Например бд при поднятии контейнера не сразу готова к работе. Эффективно работает в связке с depends_on.
### 9.2 Создаем docker-compose файл.
Создадим файл docker-compose.yaml и перенесем в него инструкции докера, которыми поднимали контейнеры ранее:  
```
version: '3'

services:
  db:
    image: 7_database
    container_name: database
    environment:
      POSTGRES_DB: docker_app_db
      POSTGRES_USER: docker_app
      POSTGRES_PASSWORD: docker_app
    volumes:
      - todo_db:/var/lib/postgresql/database
    networks:
      - todo_net

  backend:
    image: 7_back
    environment:
      HOST: db
      PORT: 5432
      DB: docker_app_db 
      DB_USERNAME: docker_app 
      DB_PASSWORD: docker_app     

  nginx:
    image: 7_nginx
    ports:
      - 80:80
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro # указываем путь от папки с файлом docker-compose.yaml

volumes:
  todo_db:
```
Запустим контейнеры командой:
```
docker-compose up -d
```
Все работает.
### 9.3 Расширяем docker-compose файл.
Добавим проверки к нашим контейнерам. Начнем с базы данных.  
Возможен вариант, когда мы подняли контейнер, но он не работает, вылетает ошибка. Ранее мы это никак не отлавливали.
Добавим инструкции:  
```
  db:
    image: 7_database
    container_name: database
    ...
    restart: always # поднимет упавший контейнер с бд
    healthcheck:
      test: ["CMD-SHELL", "pg_isready", "-U", "docker_app"] # pg_isready - внутр. команда postgres, -U - пользователь, "docker_app"- имя пользователя.
      interval: 5s
      timeout: 5s
      retries: 3
```
Поднимем контейнеры и выведем их список (docker ps). Возле контейнера database видим пометку healthy - контейнер работает нормально.
Давайте при запуске контейнера с бэкендом делать проверку, что бд функционирует без ошибок. Дополним docker-compose файл.

```
 backend:
    image: 7_back
    ...
    healthcheck: # запрос за ручку "тест" покажет работает ли бэкенд
      test: ["CMD", "curl", "--fail", "localhost:8000/test"] 
      interval: 5s
      timeout: 5s
      retries: 3
    
    depends_on:
      db:
        condition: service_healthy # бекенд поднимется, когда БД успешно пройдет проверку
```

Так же добавим зависимости для контейнера с фронтендом:  
```
nginx:
    image: 7_nginx
    ...
    depends_on:
      db:
        condition: service_healthy
      backend:
        condition: service_healthy
```
### 9.4 Сокращаем docker-compose файл.
Не обязательно указывать сеть, вольюмы и имена контейнеров. Докер-компоуз сгенирирует их самостоятельно. После остановки сеть и контейнеры будут удален, а вольюмы останутся. Если имя контейнера генерируется докером, то в HOST бэкенда вместо него нужно указать имя сервиса с базой данных (db).
Если нагрузка на приложение возрасла и нужно подключить дополнительные базы данных, можно сделать это инструкцией   
```
 db:
    image: 7_database
    container_name: database
    ...
    deploy:
          replicas: 3 # поднимем 3 контейнера с бд
```

Итоговый докер-компоуз файл примет вид:
```
version: '3'

services:
  db:
    image: 7_database
    environment:
      POSTGRES_DB: docker_app_db
      POSTGRES_USER: docker_app
      POSTGRES_PASSWORD: docker_app
    volumes:
      - todo_db:/var/lib/postgresql/database
    restart: always

    healthcheck:
      test: ["CMD-SHELL", "pg_isreade", "-U", "docker_app"]
      interval: 5s
      timeout: 5s
      retries: 3
    deploy:
      replicas: 3

  backend:
    image: 7_back
    environment:
      HOST: db
      PORT: 5432
      DB: docker_app_db 
      DB_USERNAME: docker_app 
      DB_PASSWORD: docker_app 
    healthcheck:
      test: ["CMD", "curl", "--fail", "localhost:8000/test"]
      interval: 5s
      timeout: 5s
      retries: 3
    depends_on:
      db:
        condition: service_healthy     

  nginx:
    image: 7_nginx
    ports:
      - 80:80
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro # ./nginx.conf
    depends_on:
      db:
        condition: service_healthy
      backend:
        condition: service_healthy
volumes:
  todo_db:
```

Кстати, для применения изменений в файле docker-compose.yaml не обязательно останавливать контейнеры. Можно перезапустить их командй `docker-compose up -d`






