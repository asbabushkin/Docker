## 8.2
```
stages:
    - style
    - test
linters:
    stage: style
    script:
        - isort --check --diff src/*
        - flake8 src/*
test:
    stage: test
    script: 
        - pytest src/test.py
```

## 8.3
```
x-eukaryotes:
    &eu-organelles
    organelles:
        - nucleus
        - mitochondria
        - endoplasmic reticulum
        - Golgi apparatus
        - ribosomes
        - cytoskeleton
```

