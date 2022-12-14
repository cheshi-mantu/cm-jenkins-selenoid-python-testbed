# Сборка тестового стенда на Jenkis + python, Selenoid, Selenoid UI, Chrome browser

## Предварительные условия

1. Железный сервер или виртуальная машина
2. Приложение эмулятор терминала
3. VSCode (текстовый редактор, правка удаленных файлов)

Пример делается на Ubuntu и в целом должен работать для debian линуксов.

### Windows

Нормальный терминал и команды линукс работают в приложении GitBash, которое входит в состав [Git for Windows.](https://git-scm.com/download/win)

## Подготовка

Работаем с аутентификацией по паре публичный + приватный ключ.

### Как работать с ssh на локальной машине

Для Windows, например, запускаем git bash на локальной windows машине, для макос — терминал

В git bash проверяем, есть ли у нас папка .ssh и что в ней лежит

```bash
ls -l ~/.ssh | grep id_rsa
```

Если пусто, то ключей у нас нет. Если ключи есть то увидим, что в папке лежат `id_rsa`, `id_rsa.pub`

Создадим пару ключей

```bash
ssh-keygen -t rsa -b 4096 -C `<email.address@email.xyz>`
```

`<email.address@email.xyz>` замените на что-то свое. Это комментарий, который вам позволит понять, к чему относится этот ключ.

Проверим, работает ли у нас ssh агент

```shell
eval $(ssh-agent -s)
```
Должны увидеть Process ID (PID) агента.

Добавим ключи ssh агенту

```shell
ssh-add ~/.ssh/id_rsa
```

Теперь с ними можно работать.

### Как добавить публичный ключ на удаленную машину

Если это хостинг, то, как правило, есть вариант скопировать туда публичный ключ и он везде будет добавляться.

Если нет, то копируем наш ключ на удаленную машину:

```shell
ssh-copy-id root@host-address-or-IP
```

### файл конфигов на локальной машине

Зачем: немного сэкономить время на вводе команд, чтобы вместо `ssh root@192.168.1.10` использовать `ssh test-bed`.

Лейбл запомнить легче, чем IP адрес.

заходим в папку с файлами ssh и запускаем VSCode. 

```shell
cd ~/.ssh
code .
```

если нет файла config, то создаем его, если есть — открываем

добавляем запись

```shell
Host test-bed
    Hostname 192.168.1.10
    User root
    Port 22
```

Теперь к удаленному серверу можно подключиться набрав в терминале `ssh test-bed`

## Начальные действия на удаленной машине

### Обновить репы, обновить бинарники, установить MC

```shell
apt update && apt upgrade -y && apt install mc
```

### Какая командная оболочка используется

```shell
echo $0
```

#### Поменять командную оболочку

Если не устраивает та, что используется по умолчанию.

```shell
chsh -s /bin/bash
```

### Установить докер

Самый ленивый способ — скриптом с сайта

```shell
curl -sSL https://get.docker.com | sh
```

## Готовим стенд

## Папочки

Переходим в /opt, клонируем текущий репозиторий, переобзываем это дело

```shell
cd /opt
git clone https://github.com/cheshi-mantu/cm-jenkins-selenoid-python-testbed.git
mv cm-jenkins-selenoid-python-testbed test-bed
cd test-bed
```

## Что в папках и зачем

`./image/Dockerfile`

Файл описывает, как нам надо поменять образ докера с дженкинсом, который мы хотим использовать.

Мы внутри добавляем питон, а также питоньи библиотеки.

`./init/selenoid`

Содержит файл `browsers.json` с описанием образов браузеров (название, версия, ссылка на образ), которые будут использоваться селеноидом.

`./work/jenkins` — рабочая папка дженкинса, куда будет происходить маппинг рабочих папок деженкинса внутри контейнера. Просто для удобства, чтобы иметь к ним доступ с локальной машины.
`./work/selenoid` — рабочая папка селеноида, куда будет происходить маппинг рабочих папок селеноида внутри контейнера.Просто для удобства, чтобы иметь к ним доступ с локальной машины.

`./docker-compose.yml` — описание сервисов и их настроек.

### Особенности сервиса Jenkins

Т.к. мы хотим, чтоб образ с дженкинсом еще и позволял нам гонять питонские тесты без агентов с питоном, мы должны такой образ собрать прямо на месте. Это делается так:

Удаляем (комментируем ссылку на исходный образ)

```yaml
#image: jenkins/jenkins:lts
```

добавляем директиву build

```yaml
    build:
      context: ./image
```

Компоуз возьмет Dockerfile из папки ./image соберет новый образ и создаст контейнер на его основе.

## Образы браузеров

Образы браузеров для Selenoid не скачиваются автоматически. Чтоб скачать те образы, которые мы захотели использовать на стенде (см. ./init/selenoid/browsers.json) надо их явно скачать, используя команду докера `pull`.

Ну, и это можно сразу же автоматизировать

Создайте файл `getchrome.sh`, сделайте его исполняемым `chmod +x getchrome.sh` и добавьте туда такие строки:

```shell
CHROME_RELEASES="100 101 102"

for RELEASE in $CHROME_RELEASES
do
    echo "Pulling chrome ${RELEASE}.0"
    docker pull selenoid/vnc:chrome_${RELEASE}.0
done
```

## Запуск стенда

```shell
docker compose up -d
```

## Настройки Jenkins

### Статус сервисов

После того, как деплой стартанул, проверяем состояние сервисов:

```shell
docker compose ps
```

Все сервисы должны быть в состоянии **Running**

### Начальный пароль администратора Jenkins

Смотрим логи Jenkins

```sheell
docker compose logs jenkins
```

В логах мы ищем что-то похожее:

```bash
jenkins_1      | Jenkins initial setup is required. An admin user has been created and a password generated.
jenkins_1      | Please use the following password to proceed to installation:
jenkins_1      |
jenkins_1      | 32d64223e62945fe8c4c1ed050360700
jenkins_1      |
jenkins_1      | This may also be found at: /var/jenkins_home/secrets/initialAdminPassword
jenkins_1      |
jenkins_1      | *************************************************************
jenkins_1      | *************************************************************
jenkins_1      | *************************************************************
```

Заходим на наш сервер в браузере, используя порт, указанный в настройках Jenkins в файле docker-compose.yml

```yaml
services:
  jenkins:
<snip>
    ports:
      - 8888:8080
<snip>
```

порт используется тот, что слева (слева — наружу, справа — тот, что внутри контейнера).

```shell
http://192.168.1.10:8888
```

И в открывшемся окне вводим пароль, который мы выудили в логах Jenkins.

### Установка плагинов, админ

В открывшемся окне выбираем установку "рекомендуемых" плагинов. Так будет быстрее, чем по одному выбирать из обширнго списка. Легче удалить ненужное потом.

Далее вводим (лучше не пропускать) данные нового администратора.

И в общем-то всё.

Jenkins готов к дальнейшей работе.

## Selenoid

### Проверить работоспособность

Зайти на http://`<ip-address-or-server-name>`:4444/status

Ожидаем, что увидим JSON ответ от приложения со списком браузеров.

## Selenoid-UI

### Проверить работоспособность

Зайти на http://`<ip-address-or-server-name>`:8080

Ожидаем, что увидим браузеры, можно попробовать запустить контейнер с браузером

### Добавить аллюровский плагин

### Настроить пайплайн

Не забыть, что нужно пробросить `REMOTE_DRV` в переменной окружения.

#### Через файл .env и echo

```groovy
pipeline {
    agent any

    stages {
        stage('Hello') {
            steps {
                git branch: 'main', url: 'https://github.com/cheshi-mantu/cm-py-demoqa-remote-wdr.git'
            }
        }
    stage('set-env') {
        steps {
            sh 'echo REMOTE_DRV=http://<IP_ADDR>:4444/wd/hub > .env'
        }
    }

    stage('run-python-tests') {
        steps {
                catchError(buildResult: 'UNSTABLE', message: 'uh oh', stageResult: 'UNSTABLE') {
                    sh 'pytest tests --alluredir=allure-results'
            }
        }
    }
        stage('reporting') {
        steps {
            allure includeProperties: false, jdk: '', results: [[path: 'allure-results']]
        }
    }
    }
}
```

#### Через файл .env и его создание

```groovy
pipeline {
    agent any

    stages {
        stage('code checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/cheshi-mantu/cm-py-demoqa-remote-wdr.git'
                writeFile encoding: 'UTF-8', file: '.env', text: 'REMOTE_DRV=http://<IP_ADDR>:4444/wd/hub'
            }
        }
        stage('running python tests') {
            steps {
                catchError(buildResult: 'UNSTABLE', message: 'uh-oh', stageResult: 'FAILURE') {
                        sh 'pytest tests --alluredir=allure-results' 

            }
        }
    }
        stage('generate allure repport') {
            steps {
                allure includeProperties: false, jdk: '', results: [[path: 'allure-results']]
            }
        }
    }
}
```


#### Через withEnv

```groovy
pipeline {
    agent any

    stages {
        stage('Hello') {
            steps {
                git branch: 'alt-site', url: 'https://github.com/cheshi-mantu/cm-py-demoqa-remote-wdr.git'
            }
        }
    stage('run-python-tests') {
        steps {
            withEnv(['REMOTE_DRV=http://<ADDRESS>:4444/wd/hub']) {
                catchError(buildResult: 'UNSTABLE', message: 'uh oh', stageResult: 'UNSTABLE') {
                    sh 'pytest tests --alluredir=allure-results'
                }
            }
        }
    }
        stage('reporting') {
        steps {
            allure includeProperties: false, jdk: '', results: [[path: 'allure-results']]
        }
    }
    }
}
```
