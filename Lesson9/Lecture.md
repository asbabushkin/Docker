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
  ![Container commands](https://github.com/asbabushkin/Docker/blob/main/Lesson9/Container%20commands.png)


