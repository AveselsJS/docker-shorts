# Выжимка по работе с Docker

Подноготная структуры Docker и подробно о всех основных функциях собрана в одном месте

## Оглавление
- [Выжимка по работе с Docker](#выжимка-по-работе-с-docker)
  - [Оглавление](#оглавление)
  - [Архитектура Docker](#архитектура-docker)
    - [Отличие Docker от Виртуальных машин](#отличие-docker-от-виртуальных-машин)
    - [Подробная схема работы](#подробная-схема-работы)
  - [Управление контейнерами](#управление-контейнерами)
    - [Жизненный цикл контейнера](#жизненный-цикл-контейнера)
    - [Сигналы процессам](#сигналы-процессам)
    - [Основные команды Docker](#основные-команды-docker)
    - [Логи Docker](#логи-docker)
    - [Команды внутри контейнера](#команды-внутри-контейнера)
  - [Docker Image](#docker-image)
    - [Подробно про Image](#подробно-про-image)
    - [Комманды работы с Image](#комманды-работы-с-image)
    - [Dockerfile](#dockerfile)
    - [Команды dockerfile](#команды-dockerfile)
  - [Сети Docker](#сети-docker)
  - [Docker volumes](#docker-volumes)
  - [Docker-compose](#docker-compose)
  - [Docker registry](#docker-registry)

## Архитектура Docker

### Отличие Docker от Виртуальных машин

Простыми словами ключевое отличие это отстутсвие у Docker гостевой OS. Docker ничего общего не имеет с виртуальными машинами, разве что только изолированность. 

[![Docker-vs-VM.jpg](https://i.postimg.cc/wv5KBDG2/Docker-vs-VM.jpg)](https://postimg.cc/zbf2tH5H)

Вся суть в процессах работы Linux. К примеру при запуске работы приложения на Nodejs первым делом запускается новый процесс с помощью внутреннего вызова комманды `fork`, который создаст новый процесс, при этом скоппировав предидущий процесс. Поэтому будет `Bash` потом `fork` потом второй `Bash` после этого подменяется новым процессом. Это делается с помощью комманды `execv` это заменит текущий процесс замененого `Bash` на текущий процесс ноды. У этого процесса появится свой process ID и он будет уже функционировать уже в рамках этого process ID. 

Этот процесс он при этом не является изолированным. Поскольку если другой процесс занимает ту же часть диска, то удалит текущий процесс. Эта проблема изорилованности процессов была до тех пор, пока в Linux не появилась команда `chroot` - эта комманда позволяет получить прообраз изоляции с помощью изменения root директории. В Docker используется конечно же не `chroot`, там используются `namespaces`. Но это некая аналогия, которая показывает как работает Docker. Запускается процесс, потом ему изменяют root директорию на какую-то кастомную и он не будет видеть никакие другие соответствующие бинарники, папки и прочее внутри своего изолированного пространства. Он также может скопировать необходимые другие бинарники и запускатся уже с ними.

[![Docker-container-namespace.jpg](https://i.postimg.cc/VNYnTwcN/Docker-container-namespace.jpg)](https://postimg.cc/cvbvgVCq)

Когда запускается новый процесс, то Docker открывает новый `namespace`. Фактически, когда мы запускаем контейнер, мы запускаем новый `Namespace`. Он имеет несколько наборов, так званные СGroups. Которые управляют ограничениями по памяти. IPC - это управления процесами, которые позволяют создавать изолированость и общение в namespace. Свой network. Mount который говорит какие директории доступны, которые доступны и какие нет. Process ID, который может повторятся с process ID текущего, то есть при запуске на хосте, process ID является уникальным, то в namespace process ID может быть свой. User, который может быть своим по аналогии с process ID. 

Кроме этого в Docker существуют некоторые обвязки, которые обвязывают все эти параметры namespace и непосредственно какие-то Vоlume и т.д.

Поэтому по сути контейнер это некая изолированная часть, изолированный namespace, который запускается на ядре хостовой машины и функционирует полностью изолировано кроме того, что оно получает доступ к ресурсам этой машины. Благодаря этой изоляции мы получаем полностью изолированное пространство в котором мы можем делать ряд различных вещей и кроме того можно запускать разные библиотеки, тем самым на одном ядре может например находится Gem Linux, а на другом namespace - Ubuntu. 

**Поэтому контейнер это не виртуальная машина, это изолированный namespace с дополнительными обвязками Docker в котором у нас запускается приложение, запускаются различные библиотеки с соответствующим ядром и после этого этот контейнер стартует и позволяет с ним работать с помощью удобного API.**

[![Docker-Engine.jpg](https://i.postimg.cc/QMxS71CS/Docker-Engine.jpg)](https://postimg.cc/XrmwSZwy)

Сам Docker не имеет ядра и поэтому мы не можем запускать процессы для одной архитектуре и запускать на другой архитектуре. То есть Docker не позволяет делать эмуляцию между процессами, поэтому если контейнер был запущен на архитектуре х64, то он успешно откроется только на такой архитектуре, а не на армовской или ещё какой-либо другой. 

Внутри Docker состоит из трёх частей и когда, например, вводится комманда `Docker ps` то это работа не с самим docker'ром, а с клиентом внутри docker'а. Это просто удобная СLI утилита. 


### Подробная схема работы

[![image.jpg](https://i.postimg.cc/7PpBR5tY/image.jpg)](https://postimg.cc/3WFC4xHP)

У нас есть клиент - тот самый клиент из которого вводятся комманды по типу `docker рs` и всё что угодно. После этого делается запрос к API к хосту, на этом хосте крутится Docker daemon, Docker daemon проверяет есть ли у нас такой image внутри, в наличии локально. Если такой image нету, то daemon пошел скачивать его из общего регистра, например пошел на Docker hub. После этого он скачивает нужный image и запускать новый контейнер. На самом деле всё что он делает это создаёт новый namespace и передаёт туда этот Image. Кладёт туда нужные либы и распаковывает этот image, чтобы всё запустить. 

## Управление контейнерами

`Docker container` - это сущность отвечающая за работу с контейнерами.  <br/>
Но название `container` можно опускать, поскольку в большинстве комманд он по умолчанию устроен под container, поскольку при работе с Docker очень приходится работать именно с container и поэтому команды работы с контейнерами вынесены на верхний уровень. Из-за этого следующие команды равны по своей сущности:  <br/>
`docker start` - `docker container start` <br/>
`docker stop` - `docker container stop`

### Жизненный цикл контейнера

Когда мы делаем `docker run` у нас скачивается `image`. Этот `Image` запускается и превращается в контейнер. После того как мы запустили контейнер у нас появляется жизненный цикл контейнера:

[![image.jpg](https://i.postimg.cc/vBhnvg6m/image.jpg)](https://postimg.cc/47Knf3PC)

Когда мы запустили контейнер, у него появляется жизненный этап `running`. Запускается наше приложение и контейнер работает.
Когда нужно остановить контейнер то необходимо использовать команду `docker stop`, которая приводит контейнер в состояние - `stopped`. 
Когда нужно контейнер уничтожить контейнер, то необходимо использовать команду `docker kill`. 
Когда нужно перезагрузить контейнер, то нужна команда `docker restart`. 
Когда нужно поставить контейнер на паузу, то нужна команда `docker pause`.
Если нужно удалить контейнер, то используется команда `docker rm` 
Если нужно удалить контейнер, даже если он запущен, тогда необходимо применить команду `docker rm -f`.

### Сигналы процессам

Каждая команда сопровождается сигналом, который позволяет нам завершить процесс:
| Комманда Docker | Сигнал | Пояснение |
| :-------------: | :----- | :-------- |
| docker stop | SIGTERM <br/> SIGKILL | При введении команды, Docker'у посылается команда SIGTERM, которая завершает процесс, но если в течении какого-то времени контейнер не останавливается, то через какое-то время Docker с помощью сигнала SIGNKILL убьёт этот контейнер. То есть даже если контейнер повис, Docker всё равно его убьёт |
| docker pause | SIGNSTOP | Ставит контейнер на паузу |
| docker kill | SIGNKILL | Убивает контейнер, даже если он повис |

### Основные команды Docker

| Комманда | Синтаксис | Пояснение |
| :------: | :-------- | :-------- |
| run --name | docker run --name <container_name> | Создание контейнера с указанным именем | 
| start | docker container start <container_name> | Запуск контейнера с указанным именем | 
| stop | docker container stop <container_name> | Остановка контейнера с указанным именем | 
| ps -a | docker ps -a | Вывести все (как запущенные так и остановленные) контейнеры | 
| ps  | docker ps | Вывести все запущенные контейнеры | 
| remove | docker container remove <container_name> | Удалить контейнера с указанным именем | 
| prune | docker container prune | Удалить все остановленные контейнеры | 
| rename | docker remae <old_container_name> <new_container_name> | Переименовать контейнер с указанным именем на новое имя |
| stats | docker stats | Выводит статистику по всем контейнерам в реальном времени таких параметров как: ЦПУ, использование памяти, выход и выход сети, ID процесса и т.д. |
| inspect | docker inspect <container_name> | Получить всю подробную информацию о контейнере в формате JSON, в том числе State, ID процесс или Image`и с которых был сделан контейнер |
| inspect -s | docker inspect -s <container_name> | Получить размер контейнера |
| inspect -f "{{.field}}" | docker inspect -f "{{.field.field}}" <container_name> | Получить детали конкретной строчки из JSON |

### Логи Docker

Контейнер на протяжении своего жизненного цикла создаёт логи. Чтобы получить все логи конкретного контейнера необходимо ввести комманду:
```
docker logs <container_name>
```

Поскольку лента логов достаточно длинная и поэтому слабо читабельная, то уже с помощью команд Linux есть возможность витянуть из логов что-то конкретное. 
Для этого необходимо использовать оператор пайпа ( | ), чтобы передать результаты предидущей команды в следующую команды, а также использовать функцию `grep`. Функция `grep` позволяет вытащить необходимый кусок текста описывая регулярные выражения в RegExp внутри запроса. 

| Синтаксис | Пояснение |
| :-------- | :-------- |
| docker logs <container_name> \ grep `id` | Выводит все строки содержащие id |
| docker logs <container_name> \ grep `id` -A 10 | Выводит 10 строк после нахождения строки содержащую id |
| docker logs <container_name> \ grep `id` -В 15 | Выводит 15 строк до нахождения строки содержащую id |
| docker logs <container_name> \ grep `id` -m 2 | Выводит 2 первых строки содержащие id |
| docker logs <container_name> text.txt | Сохранить логи, которые будут выведены в файл text.txt |

### Команды внутри контейнера 

Порой необходимо войти в контейнер руками, чтобы что-нибудь сделать, подправить или посмотреть. Синтаксис docker команда для контейнера:
```
docker exec [параметры] <container_name> [комманда]
```

При этом параметры могут быть следующими: 
| Параметр | Пояснение |
| :------: | :-------- |
| -i | итерактивное |
| -t | псевдо tty |
| -d | запуск в фоне |
| -e | переменная окружения |
| -u | пользователь |
| -w | рабочая директория |

А комманды:
| Комманда | Пояснение |
| :------: | :-------- |
| bash | исполнить код |
| pwd | выводит текущую рабочую директорию |

Любое исполнение команд сработает, если у контейнера статус `running`, другими словами – контейнер запущен.

## Docker Image

### Подробно про Image

Image из себя представляет некий список слоёв. Каждый из этих слоёв имеет свой уникальный идентификатор. Это поможет сильно экономить пространство на диске. 

[![Image.jpg](https://i.postimg.cc/YCDKh1zf/Image.jpg)](https://postimg.cc/NKXPn2V5)

У нас есть некий контейнер, который (впрочем как и все) состоит из слоёв. Каждый слой на самом деле это тоже Image. Это некий слепок, который содержит некую информацию. Все слои, которые содержит наш Image доступны нам только на чтение. То есть когда строится наш контейнер, он сформировывает 6 слоёв. После того как создастся наш контейнер, создастся ещё один, тонкий слой и вот этот слой доступен на запись. Поэтому сколько бы мы одинаковых контейнеров не запускали, они все будут базироватся на одном Image таким образом сильно экономя пространство. 

[![image.jpg](https://i.postimg.cc/3Rs3Wg7b/image.jpg)](https://postimg.cc/MMmCFfny)

Каждый Image имеет конкретный этап сборки и при запуске двух одинаковых контейнеров, будет накладыватся лишь один, отличающийся слой, при этом та часть, что доступна для чтения будет как раз общая. 

### Комманды работы с Image

Все комманды можно получить с помощью комманды --help

| Команда | Описание | Пример комманды |
| :-----: | :------- | :-------------- |
| pull | Выкачать Image с Docker Hub | docker pull <image_name> |
| images | Просмотреть все Image'и | docker images | 
| save |  Сохранить Image на диск | docker save --output <image_name_archive> <image_name> |
| history | Просмотреть как был собран образ | docker history <image_name> |
| build | Построить image из dockerfile | docker build |
| import | Развернуть образ из архива | docker image import <image_archive_name> |
| inspect | Получить подробную информацию по самому образу: когда был создан, его ID, название контейнеров и так далее | docker inspect <image_name> |
| ls | Вывести все текущие образы, которые есть. Также работает динамическая строка форматирования | docker image ls |
| prune | Чистит неактивные образы или образы у которых отсутствует тэг | docker image prune |
| push | Пушит сбилденный образ в registry | docker push <image_name> |
| rm | Удаляет конкретный образ | docker image rm <image_name> |
| save | Сохранить образ в некоторый архив, чтобы потом куда-нибудь его перекинуть | docker image save <image_name> |
| tag | Создать тэг для контейнера | docker tag <tag_name> |

### Dockerfile

Dockerfile состоит из строк, где каждая строка начинается из какой-то команды и её аргументов. Например WORKDIR /opt/app означает: 
- WORKDIR комманда
- /opt/app - аргумент


[![Dockerfile.jpg](https://i.postimg.cc/J7kQ0gX4/Dockerfile.jpg)](https://postimg.cc/YjtFyd8J)

Каждый раз когда добавляется строка, добавляется новый слой. Поэтому необходимо оптимизировать Dockerfile. При этом есть ограничение количества строк в размере 127.
Помимо постройки image с dockerfile есть ещё такое понятие как контекст. Контекст это путь выполнения команды `docker build` со всеми вложенными директориями. Поэтому если указать начальную папку как корневую, то контекстом будет вся операционная система это будет медленно и неефективно. Поэтому важно понимать, где начинается контекст. Там где производится `docker build` там и контекст. 

При этом мы не можем выйти вверх. Поэтому если нам нужны какие-то файлы из верха, то нам нужно начинать строить приложение из этого верха. Также из контекста можно что-то удалить с помощью файла `.dockerignore` например `node_modules` как например в `.gitignore`

### Команды dockerfile

[![dockerfile.jpg](https://i.postimg.cc/GhPJJyzW/dockerfile.jpg)](https://postimg.cc/ZCRdJ0GH)

Подробнее о командах dockerfile:
- Аргументы это дополнительные параметры, которые можно передать при сборке. Аргументы никогда не остаются после того, как был собран image. Это аргументы, которые распростроняются во время билда. В финальном сборке их не будет. Это для передачи из вне. Это полезно например, когда билдится какой-нибудь нодовский модуль и нужен приватный репозиторий и туда передать какой-то токен, который должен существовать только в рамках билда и в рамках этого билда для того, чтобы установить зависимости.
- `FROM` это то на чём базируется image. Он должен быть обязателен при каждой сборке. Также этому image можно задать алиас, чтобы сделать multistage билдинг.
- `ONBUILD` - это полезно только тогда, когда строится какой-то image, который базируется на другом образе.
- `LABEL` - это мета информация. Это та информация, которая останется в финальной сборке и её можно будет посмотреть.
- `USER` и `WORKDIR` это информация о пользователе, который будет иметь доступ к рабочей папки. Всё что до строчки с `USER` выполняется с дефолтным пользователем в корневой папке, а всё что ниже - выполняется уже с привязанным пользователем. 
- Команда `ADD` чаще используется когда нужно добавить что-то из хостовой машины в контейнер сборки. Например это может быть гит репозиторий. Первым аргументом он принимает то, **что нужно скопировать**, а вторым аргументом **куда это нужно скопировать**. В отличии от `COPY` способен делать дополнительные фичи, такие как разархивация архива или например скачать архив с URL.
- Команда `COPY` в отлчии от `ADD` позволяет нам коппировать инфорацию из других image при multistage build.  

[![Dockerfilee.jpg](https://i.postimg.cc/sgNBpfdF/Dockerfilee.jpg)](https://postimg.cc/Bj2Q30nN)

- `SHELL` - позволяет также как и `USER` и `WORKDIR` установить какой shell будет использоваться после этой строки. 
- `RUN` - позволяет выполнить какую-то комманду в `SHELL` например установка каких-то зависимостей из репозитория. 
- `ENV` - позволяет объявить и использовать их несколько раз. Если нам нужно передать какие-то переменные, но при этом чтобы эти переменные не передались в финальную сборку, нужно передать эти команды перед самой командой. 
- `STOPSIGNAL` - позволяет отправить специализированный сигнал. 
- `EXPOSE` - это просто некоторая документация для тех, кто будет использовать этот image о том, что можно будет прокинуть порт 80 по TCP и в него чем-то там дёрнуть и будет получено то что там спрятано. 
- `#` - комментарии.

## Сети Docker

## Docker volumes

## Docker-compose

## Docker registry



