## Лекция 8. YAML и Docker-compose.
### 8.1. YAML.
На практике нам редко приходится работать с одним-двумя контейнерами. Как правило, приложения разрастаются, масштабируются и требуют значительных усилий для организации работы сети контейнеров. Требуетя программа для управления многоконтейнерными приложениями и это - Docker Compose. Логика разворачивания сети контейнеров описывается в файле docker-compose.yaml. 
Yaml можно назвать языком хранения данных. Основными элементами являются пары ключ-значение, отступы и последовательности (аналог списков).  
Примеры представления данных yaml-файлах:  
```
# Ключ и два значения через пробел:
name: Peter Olga
# Result: {'name': 'Peter Olga'}

# Ключ и два значения через пробел (2-й вариант):
name:
    Peter 
    Olga
# Result: {'name': 'Peter Olga'}

# Ключ и два значения через перенос строки:
name: |
    Peter 
    Olga
# Result: {'name': 'Peter\nOlga\n'}

# Параграф:
name: >
    Peter 
    Olga

# Список:
name:
    - Peter 
    - Olga
# Result {'name': ['Peter', 'Olga']}

# Список словарей:
name:
    - Peter: 
      Olga:
# Result: {'name': [{'Peter': None}, {'Olga': None}]}

# Поместить несколько словарей в один список:
name:
    - Peter: 12
      Olga: 13

    - Oleg: 7
      Julia: 2
# Result: {'name': [{'Peter': 12, 'Olga': 13}, {'Oleg': 7, 'Julia': 2}]}
```
Для повторяющихся элементов следует использовать синтаксис & и <<: * - это якорь и ссылка на якорь:
```
# Повторяющиеся элементы:
team:
    backend:
        - Peter:
            position: junior
            salary: 55000
        - Olga:
            position: junior
            salary: 55000

# Следует описывать так:

junior:
    &junior   # якорь с именем junior
    position:junior
    salary:55000

team:
    backend:
        - Peter:
            <<: *junior   # ссылка на якорь
        - Olga:
            <<: *junior   # ссылка на якорь
```
