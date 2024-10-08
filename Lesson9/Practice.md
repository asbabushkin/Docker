## 9.1
```
version: '3'

services:
    database:
        image: postgres:14
        environment:
            POSTGRES_DB: todo_list
            POSTGRES_USER: admin
            POSTGRES_PASSWORD: admin
        volumes:
            - postgres:/var/lib/postgresql/data

        healthcheck:
            test: ["CMD-SHELL", "pg_isready", "-U", "admin"]

    create-table:
        image: postgres:14
        command: bash -c 'PGPASSWORD=admin psql -U admin --dbname todo_list -p 5432 -h database -c "CREATE TABLE IF NOT EXISTS user_table (user_id int PRIMARY KEY, username varchar(256), email varchar(256));"'

        depends_on:
            database:
                condition: service_healthy

volumes:
  postgres:
```
## 9.2
```
version: '3'

services:
    database:
        image: postgres:14
        environment:
            POSTGRES_DB: todo_list
            POSTGRES_USER: admin
            POSTGRES_PASSWORD: admin
        volumes:
            - postgres:/var/lib/postgresql/data

        healthcheck:
            test: ["CMD-SHELL", "pg_isready", "-U", "admin"]

    backend:
        image: kcoursedocker/task-9.2-back:latest
        environment:
            PG_NAME: todo_list
            PG_USER: admin
            PG_PASSWORD: admin
            PG_HOST: database
        depends_on:
            database:
                condition: service_healthy

    frontend:
        image: kcoursedocker/task-9.2-front:latest
        ports:
            - 127.0.0.1:80:80
        depends_on:
            database:
                condition: service_healthy
            backend:
                condition: service_started        

    backend-migrations:
        image: kcoursedocker/task-9.2-back:latest
        environment:
            PG_NAME: todo_list
            PG_USER: admin
            PG_PASSWORD: admin
            PG_HOST: database
        command: bash -c "make migrate"
        depends_on:
            database:
                condition: service_healthy
            backend:
                condition: service_started

    backend-superuser:
        image: kcoursedocker/task-9.2-back:latest
        environment:
            PG_NAME: todo_list
            PG_USER: admin
            PG_PASSWORD: admin
            PG_HOST: database
            DJANGO_SUPERUSER_USERNAME: admin
            DJANGO_SUPERUSER_PASSWORD: admin
            DJANGO_SUPERUSER_EMAIL: admin@admin.com
        command: bash -c "make create-superuser"
       
        depends_on:
            database:
                condition: service_healthy
            backend:
                condition: service_started
            backend-migrations:
                condition: service_started


volumes:
  postgres:
```
