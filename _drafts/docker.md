---
layout: post
title: Как и для чего использовать Docker
subtitle: Подзаголовок (выводится на странице поста)
summary: Короткое описание (выводится на главной странице)
cover_url: "Путь к файлу-обложке для соц. сетей, например, /images/vscode_eslint.png"
---

**Docker - программа позволяющая запускать процессы операционной системе в изолированном окружении на базе специально созданных образов. Несмотря на то что технологии лежащие в основа докера появились до него, именно докер произвел революцию в том как сегодня создается инфраструктура проектов, собираются и запускаются сервисы.**

Данный гайд состоит из четырех частей. В первой устанавливается и запускается докер. Во второй объясняется его суть и польза, а так же варианты использования. В третьей описывается внутреннее устройство. В четвертой описываются некоторые из его дополнительных возможностей. Несмотря на все это, невозможно в рамках одного руководство описать все то, что умеет делать докер. За конкретными рецептами и более подробными описаниями обращайтесь к официальной документации. И идеально если вы повторите все шаги из этого гайда на своем компьютере.

```
В статье намеренно опущены многие детали, чтобы не грузить информацией, которая не нужна для ознакомления.
```

### Установка

Докер поставляется в виде Community Edition (CE) и Enterprise Edition (EE). Нас интересует CE. На [главной странице](https://www.docker.com/community-edition) этой версии доступны ссылки для скачивания под все популярные платформы. Выберите вашу и установите докер.

В работе докера есть одна деталь, которую важно знать при установке на Mac и Linux. По умолчанию, докер работает через unix сокет. В целях безопасности, сокет закрыт для пользователей не входящих в группу _docker_. И хотя установщик добавлет текущего пользователя в эту группу автоматически, докер сразу не заработает. Дело в том, что если пользователь меняет группу сам себе, то ничего не изменится до тех пор пока пользователь не перелогинится. Такова особенность работы ядра. Для проверки того в какие группы входит ваш пользователь, можно набрать команду `id`.

Проверить успешность установки можно командой `docker info`:

```sh
$ docker info
Containers: 22
 Running: 2
 Paused: 0
 Stopped: 20
Images: 72
Server Version: 17.12.0-ce
Storage Driver: overlay2
 Backing Filesystem: extfs
 Supports d_type: true
 Native Overlay Diff: true
Logging Driver: json-file
Cgroup Driver: cgroupfs
...
```

Она выдает довольно много информации о конфигурации самого докера так и статистику работы.

### Запуск

Начнем с самого простого варианта:

```sh
$ docker run -it nginx bash
root@a6c26812d23b:/#
```

При первом вызове, данная команда начнет скачивать образ (image) _nginx_, поэтому придется немного подождать. После того как образ скачается, запустится _bash_ и вы окажетесь внутри контейнера (container). Побродите по файловой системе, посмотрите папку `/etc/nginx`. Все что вы сделаете здесь внутри, никак не затронет вашу основную файловую систему. Вернуться в родные скрепы можно командой `exit`. Теперь посмотрим вариант вызова команды `cat`:

```sh
$ docker run nginx cat /etc/nginx/nginx.conf
...
$
```

Эта команда выполняется практически мгновенно, так как образ уже загружен. В отличие от предыдущего старта, где запускается баш и начинается интерактивная сессия, запуск команды `cat /etc/nginx/nginx.conf` для образа _nginx_ выведет на экран содержимое указанного файла и вернет управление в то место где вы были. Вы не окажетесь внутри контейнера.

Последний вариант запуска будет таким:

```sh
$ docker run -p 8080:80 nginx
```

Данная команда не возвращает управление потому что стартует Nginx. Откройте браузер и наберите `localhost:8080`. Вы увидите как загрузилась страница _Welcome to nginx!_. Если в этот момент снова посмотреть в консоль где был запущен контейнер, то можно увидеть что туда выводится лог запросов к `localhost:8080`. Остановить Nginx можно командой <kbd>Ctrl + C</kbd>

Несмотря на то что все запуска выполнялись по разному и приводили к разным результатам, общая схема их работы - одна и та же. Докер при необходимости автоматически скачивает образ (первый аргумент после `docker run`) и на основе него стартует контейнер с указанной командой.

Образ - слепок файловой системы. Пока мы используем готовые, но потом научимся создавать их самостоятельно.
Контейнер - запущенный процесс операционной системы в изолированном окружении с подключенной файловой системой из образа.

Повторюсь что контейнер всего лишь обычный процесс вашей операционной системы. Разница лишь в том, что благодаря возможностям ядра (о них в конце) докер стартует процесс в изолированном окружении. Контейнер видит свой собственный список процессов, свою собственную сеть, свою собственную файловую систему и так далее. Пока ему не укажут явно, он не может взаимодействовать с вашей основной операционной системой и всем что в ней хранится или запущено.

Попробуйте выполнить команду `docker run -it ubuntu bash` и наберите `ps auxf` внутри запущенного контейнера. Вывод будет таким:

```sh
USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root         1  0.1  0.1  18240  3300 pts/0    Ss   15:39   0:00 /bin/bash
root        12  0.0  0.1  34424  2808 pts/0    R+   15:40   0:00 ps aux
```

То есть видно что процесса всего два и у Bash PID равен 1. Заодно можно посмотреть в папку `/home` командой `ls /home` и убедиться что она пустая.

### Зачем все это?

Докер можно использовать для разных целей. Ключевая - универсальная доставка и изолированный запуск любых программ. Вспомните когда вам приходилось собирать программы из исходников. Этот процесс включает в себя следующие шаги:

* Установить все необходимые зависимости под вашу операционную систему (их список еще надо найти)
* Скачать архив, распаковать
* Запустить конфигурирование `make configure`
* Запустить компиляцию `make compile`
* Установить `make install`

Как видите, процесс нетривиальный и далеко не всегда быстрый. Иногда даже невыполнимый. И это не говоря про загрязнение операционной системы.

Докер позволяет упростить эту процедуру до запуска одной команды причем с почти 100% гарантией успеха. Посмотрите на настоящий пример:

```sh
добавить пример
```

Запуск команды выше приводит к тому в основной системе, в папке `...` оказывается исполняемый файл программы LALA, которая была скомпилирована на локальном компьютере, но внутри запущенного контейнера. Теперь LALA запускается без докера командой `ehu`. Весь фокус в том, что образ, из которого был запущен контейнер, подготовлен программистами разрабатывающими LALA. Внутри него установлены все необходимые зависимости и его запуск практически гарантирует 100% работоспособность независимо от вашей основной ОС.

Часто даже не обязательно копировать программу из контейнера на вашу основную систему. Достаточно запускать сам контейнер когда в этом возникнет необходимость. Предположим что мы решили разработать статический сайт на основе Jekyll. Jekyll - популярный генератор статических сайтов написанный на руби. Например гайд который вы читаете прямо сейчас, находится на статическом сайте сгенерированном с помощью Jekyll. И при его генерации использовался докер (об этом можно прочитать в гайде: [как делать блог на Jekyll](http://guides.hexlet.io/jekyll/)).

Старый способ использования Jekyll требовал установки на вашу основную систему как минимум ruby и самого Jekyll в виде гема (название пакетов в ruby). Причем, как и всегда, в подобных вещах, Jekyll работает только с определенными версиями руби, что вносит свои проблемы при настройке. С докером, запуск Jekyll сводится к одной команде, выполняемой в папке с блогом (Подробнее можно посмотреть в [репозитории](https://github.com/hexletguides/hexletguides.github.io) наших гайдов):

```sh
docker run --rm --volume="$PWD:/srv/jekyll" -it jekyll/jekyll jekyll server
```

Точно таким же образом сейчас запускается огромное количество различного софта. Чем дальше, тем больше подобный способ захватывает мир. На этом месте можно немного окунуться в происхождение названия Docker.

![](картинка)

Как вы знаете, основной способ распространения товаров по миру - корабли. Раньше стоимость перевозки была довольно большая, по причине того, что каждый груз имел собственную форму и тип материала. Загрузить на корабль мешок с рыбой или машину - две большие разницы. Возникали проблемы со способами погрузки требовавшими разнообразные краны и инструменты. С другой стороны, эффективно упаковать груз на самом корабле, тоже не тривиальная задача.


Но в какой-то момент все изменилось. Картинка стоит тысячи слов:

![](картинка)

Контейнеры уровняли все виды грузов. Что в свою очередь привело к упрощению процессов, ускорению и следовательно удешевлению.

Ровно тоже самое произошло и в разработке. Docker стал универсальным средством доставки софта независимо от его структуры, зависимостей и способа установки. Все что нужно программам, распространяемым через докер, находится внутри образа и не пересекается с основной системой и другими контейнерами. Важность этого факта невозможно переоценить. Теперь обновление версий программ никак не задействует ни саму систему, ни другие программы. Сломаться больше ничего не может. Все что нужно сделать это скачать новый образ той программы, которую требуется обновить. Другими словами, докер убрал проблему [dependency hell](https://ru.wikipedia.org/wiki/Dependency_hell) и сделал инфраструктуру [immutable](https://martinfowler.com/bliki/ImmutableServer.html) (неизменяемой).

Больше всего, Docker повлиял именно на серверную инфраструктуру. До эры докера, управление серверами было очень болезненным мероприятием даже не смотря на наличие программ по управлению конфигурацией (chef, puppet, ansible). Основная причина всех проблем - изменяемое состояние. Программы ставятся, обновляются, удаляются. Происходит это в разное время на разных серверах и немного по разному. Например обновить версию таких языков как php, ruby или python, могло стать целым приключением с потерей работоспособности. Проще поставить рядом новый сервер и переключиться на него. Идейно докер позволяет сделать именно такое переключение. Забыть про старое и поставить новое. Докеру без разницы каким было предыдущее состояние, ведь каждый запущенный контейнер живет в своем окружении. Причем откат в такой системе тривиален. Все что нужно - остановить новый контейнер и поднять старый, на базе предыдущего образа.

### Приложение в контейнере

Теперь поговорим о том, как приложение отображается на контейнеры. Возможны два подхода:

1. Все приложение - один контейнер внутри которого поднимается дерево процессов, приложение, веб сервер, база данных и все в этом духе.
1. Каждый запущенный контейнер - атомарный сервис. Другими словами каждый контейнер представляет из себя ровно одну программу, будь-то веб-сервер или приложение.

На практике, все преимущества Docker достигаются только со вторым подходом. Во-первых, сервисы, как правило, разнесены по разным машинам и нередко перемещаются по ним (например в случае выхода из строя сервера), во-вторых, обновление одного сервиса, не должно приводить к остановке остальных.

Первый подход, крайне редко, но бывает нужен. Например Хекслет работает в двух режимах. Сам сайт с его сервисами использует вторую модель, когда каждый сервис отдельно, но вот практика выполняемая в браузере, стартует по принципу один пользователь - один контейнер. Внутри контейнера может оказаться все что угодно в зависимости от практики. Как минимум там всегда стартует сама Хекслет IDE, а она в свою очередь порождает терминалы (процессы). В курсе по базам данных в этом же контейнере стартует и база данных, в курсе связанном с вебом - стартует веб-сервер. Такой подход позволяет создать иллюзию работы на настоящей машине и резко снижает сложность в поддержке упражнений. Повторюсь что такой вариант использования очень специфичен и вам врядли понадобится.

Другой важный аспект при работе с контейнерами касается состояния. Например если база запускается в контейнере, то ее данные ни в коем случае не должны храниться там же, внутри контейнера. Контейнер как процесс, постоянно претерпевает процесс уничтожения и создания, его наличие всегда временно. Docker содержит механизмы, для хранения и использования данных лежащих в основной файловой системе. О них будет позже.

### Работа с образами

Docker - больше чем просто программа, это целая экосистема со множеством проектов и сервисов. Главный сервис, с которым вам придется иметь дело - Registry. Хранилище образов. Концептуально оно работает также как и репозиторий пакетов любого пакетного менеджера. Посмотреть его содержимое можно на сайте https://store.docker.com/ кликнув по ссылке Containers.

Когда мы выполняем команду _run_ `docker run <image name>`, то Docker проверяет наличие указанного образа на локальной машине и скачивает его по необходимости. Список образов уже скачанных на компьютер, можно посмотреть командой `docker images`.

```sh
# В выводе присутствуют образы с именем <none> подробнее о них можно прочитать здесь
$ docker images
REPOSITORY                           TAG                 IMAGE ID            CREATED             SIZE
workshopdevops_web                   latest              cfd7771b4b3a        2 days ago          817MB
hexletbasics_app                     latest              8e34a5f631ea        2 days ago          1.3GB
mokevnin/rails                       latest              96487c602a9b        2 days ago          743MB
ubuntu                               latest              2a4cca5ac898        3 days ago          111MB
ruby                                 2.4                 713da53688a6        3 weeks ago         687MB
ruby                                 2.5                 4c7885e3f2bb        3 weeks ago         881MB
nginx                                latest              3f8a4339aadd        3 weeks ago         108MB
elixir                               latest              93617745963c        4 weeks ago         889MB
postgres                             latest              ec61d13c8566        5 weeks ago         287MB
```

Разберемся с тем как формируется имя образа и что оно в себя включает. Вторая колонка в выводе выше называется TAG. Когда мы выполняли команду `docker run nginx`, то на самом деле выполнялась команда `docker run nginx:latest`. То есть мы не просто скачиваем образ _nginx_, а скачиваем его конкретную версию. Latest тег по умолчанию. Несложно догадаться что он означет последнюю версию образа. Важно понимать что это всего лишь соглашение, а не правило. Конкретный образ вообще может не иметь тега _latest_, либо иметь, но он не будет содержать последние изменения, просто потому что никто их не публикует. Впрочем популярные образы следуют соглашению. Как понятно из контекста, теги в докере изменяемы, другими словами вам никто не гарантирует что скачав образ с одним и тем же тегом на разных компьютерах в разное время, вы получите одно и тоже. Такой подход может показаться странным и ненадежным, ведь нет гарантий, но на практике есть определенные соглашения, которым следуют все популярные образы. Тег latest действительно всегда содержит последнюю версию и постоянно обновляется, но кроме кроме этого тега активно используется [семантическое версионирование](https://semver.org/). Рассмотрим https://store.docker.com/images/nginx

    1.13.8, mainline, 1, 1.13, latest
    1.13.8-perl, mainline-perl, 1-perl, 1.13-perl, perl
    1.13.8-alpine, mainline-alpine, 1-alpine, 1.13-alpine, alpine
    1.13.8-alpine-perl, mainline-alpine-perl, 1-alpine-perl, 1.13-alpine-perl, alpine-perl
    1.12.2, stable, 1.12
    1.12.2-perl, stable-perl, 1.12-perl
    1.12.2-alpine, stable-alpine, 1.12-alpine
    1.12.2-alpine-perl, stable-alpine-perl, 1.12-alpine-perl

Теги, в которых присутствует полная семантическая версия (x.x.x) всегда неизменяемы, даже если в них встречается что то еще, например _1.12.2-alphine_. Такую версию смело нужно брать для продакшен окружения. Теги подобные такому _1.12_ обновляются при измении path версии. То есть внутри образа может оказаться и версия _1.12.2_ и в будущем _1.12.8_. Точно такая же схема и с версиями в которых указана только мажорная версия, например _1_. Только в данном случае обновление идет не только по патчу, но и по минорной версии.

Как вы помните, команда `docker run` скачивает образ если его нет локально, но эта проверка не связана с обновлением содержимого. Другими словами, если _nginx:latest_ обновился, то `docker run` его не будет скачивать, он использует тот _latest_, который прямо сейчас уже загружен. Для гарантированного обновления образа существует другая команда: `docker pull`. Вот она всегда проверяет обновился ли образ для определенного тега.

Кроме тегов, имя образа может содержать префикс: `etsy/chef`. Этот префикс является именем аккаунта на https://cloud.docker.com/ (или старый сайт https://hub.docker.com/) сайте, через который создаются образы попадающие в Registry. Большинство образов как раз такие, с префиксом. И есть небольшой набор, буквально сотня образов, которые не имеют префикса. Их особенность в том, что эти образы поддерживает сам Docker. Поэтому если вы видите что в имени образа нет префикса, значит это официальный образ. Список таких образов можно увидеть здесь: https://github.com/docker-library/official-images/tree/master/library

Удаляются образы командой `docker rmi <imagename>`.

```sh
$ docker rmi ruby:2.4
Untagged: ruby:2.4
Untagged: ruby@sha256:d973c59b89f3c5c9bb330e3350ef8c529753ba9004dcd1bfbcaa4e9c0acb0c82
```

Если в докере присутствует хоть один контейнер из удаляемого образа, то докер не даст его удалить по понятным причинам. Если вы все же хотите удалить и образ и все контейнеры связанные с ним, то используйте флаг `-f`.

### Управление контейнерами

![Docker Container LifeCycle](/images/docker/docker-container-lifecycle.png)

Картинка выше, описывает жизненный цикл (конечный автомат) контейнера. Кружками на нем изображены состояния, жирным выделены команды консольные команды, а квадратиками показывается то что в реальности выполняется.

Проследите путь команды `docker run`. Несмотря на то что комнада одна, с точки зрения работы докера выполняется два действия. Создание контейнера и запуск. Существуют и более сложные варианты исполнения, но в этом разделе мы рассмотрим только базовые команды.

Запустим nginx так, чтобы он работал в фоне. Для этого после слова _run_ добавляется флаг `-d`:

```sh
$ docker run -d -p 8080:80 nginx
431a3b3fc24bf8440efe2bca5bbb837944d5ae5c3b23b9b33a5575cb3566444e
```

После ее выполнения докер выводит идентификатор контейнера и возвращает управление. Убедитесь в том что Nginx работает открыв в браузере ссылку `localhost:8080`. В отличие от предыдущего запуска, наш Nginx работает в фоне, а значит не видно его вывода (логов). Посмотреть его можно командой `docker logs`, которой нужно передать идентификатор контейнера:

```sh
$ docker logs 431a3b3fc24bf8440efe2bca5bbb837944d5ae5c3b23b9b33a5575cb3566444e

172.17.0.1 - - [19/Jan/2018:07:38:55 +0000] "GET / HTTP/1.1" 200 612 "-" "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_13_2) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/63.0.3239.132 Safari/537.36" "-"
```

Вы так же можете подсоединиться к выводу лога в стиле `tail -f`. Для этого запустите `docker logs -f 431a3b3fc24bf8440efe2bca5bbb837944d5ae5c3b23b9b33a5575cb3566444e`. Теперь лог будет обновляться каждый раз, когда вы обновляете страницу в браузере. Выйти из этого режима можно набрав <kbd>Ctrl + C<kbd>, при этом сам контейнер остановлен не будет.


Теперь выведем информацию о запущенных контейнерах воспользовавшись командой `docker ps`:

```sh
CONTAINER ID        IMAGE                            COMMAND                  CREATED             STATUS              PORTS                                          NAMES
431a3b3fc24b        nginx                            "nginx -g 'daemon of…"   2 minutes ago       Up 2 minutes        80/tcp                                         wizardly_rosalind
```

Расшифровка столбиков:

* CONTANINER_ID - идентификатор контейнера. Так же как и в гит, используется сокращенная запись хеша.
* IMAGE - имя образа из которого был поднят контейнер. Если не указан тег, то подразумевается _latest_.
* COMMAND - команда, которая выполнилась на самом деле при старте контейнера.
* CREATED - время создания контейнера
* STATUS - текущее состояние.
* PORTS - проброс портов.
* NAMES - Алиас. Докер позволяет кроме идентификатора иметь имя. Так гораздо проще обращаться с контейнером. Если при создания контейнера имя не указано, то докер самостоятельно его придумывает. В выводе выше как раз такое имя у Nginx.

```
Хозяйке на заметку: Команда docker stats выводит информацию о том сколько ресурсов потребляют запущенные контейнеры.
```

Теперь попробуем остановить контейнер. Выполним команду:

```sh
# Вместо CONTAINER_ID можно указывать имя
$ docker kill 431a3b3fc24b # docker kill wizardly_rosalind
431a3b3fc24b
```

Если попробовать набрать `docker ps` то там этого контейнера больше нет, он удален.

Команда `docker ps` выводит только запущенные контейнеры. Но кроме них могут быть и остановленные. Причем остановка может происходить как и по успешному завершению так и в случае ошибок. Попробуйте набрать `docker run ubuntu ls`, а затем `docker run ubuntu bash -c "unknown"`. Эти команды не запускают долго живущего процесса, они завершаются сразу по выполнению, причем вторая с ошибкой, так как такой команды не существует. Теперь выведем все контейнеры командой `docker ps -a`. Первыми тремя строчками вывода окажутся:

```sh
CONTAINER ID        IMAGE                            COMMAND                  CREATED                  STATUS                       PORTS                                          NAMES
85fb81250406        ubuntu                           "bash -c unkown"         Less than a second ago   Exited (127) 3 seconds ago                                                  loving_bose
c379040bce42        ubuntu                           "ls"                     Less than a second ago   Exited (0) 9 seconds ago                                                    determined_tereshkova
```

Здесь как раз два последних наших запуска. Если посмотреть на колонку STATUS, то видно что оба контейнера находятся в состоянии Exited. То есть запущенная команда внутри них выполнилась и они остановились. Разница лишь в том что один завершился успешно (0), а второй с ошибкой (127). После остановки контейнер можно даже перезапустить:

```sh
docker start determined_tereshkova
```

Только в этот раз вы не увидите вывод. Чтобы его посмотреть воспользуйтесь командой `docker logs determined_tereshkova`.

И последнее в плане управления

#### Ports

#### Volumes


### Подготовка собственного образа

### Docker Compose

### В бою

### Докер под капотом

---

*Кирилл Мокевнин*
