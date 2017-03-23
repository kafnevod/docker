# docker

## Docker-образы сервисной шины `Highway Service Bus`


Стек docker-сервисов сервисной шины Highway Service Bus включает в себя два docker-образа:
* `dh.ics.perm.ru/flexberry/hwsb` — docker-образ сервисной шины Highway Service Bus;
* `dh.ics.perm.ru/flexberry/servicebuseditor` -   docker-образ редактора сервисной шины.

Кроме этого для работы с базой данных сервисной шины используется образ `dh.ics.perm.ru/kaf/alt.p8-postgresql9.6-ru` с именованным томом `postgresql-hwsb`,содержащим postgres базу данных `FlexberryHWSB` с пользователем `flexberry_orm_tester`.

### Docker-образ сервисной шины `dh.ics.perm.ru/flexberry/hwsb`

Docker-образ `dh.ics.perm.ru/flexberry/hwsb` построен на основе образа `dh.ics.perm.ru/kaf/alt.p8-mono4` с добавлением пользователя `highway` (uid:504, gid:504) и каталога `flexberry-hwsb/`, содержащего код сервисной шины. Данный каталог   располагается в каталоге `/opt/` файловой системы образа.

При запуске контейнера вызывается shell-скрипт `/opt/startFlexberry-hwsb.sh`:
```
#!/bin/sh
. /usr/bin/mono-service2 -l:/tmp/highwaysb.lock -d:/opt/flexberry-hwsb -m:highwaysb NewPlatform.Flexberry.HighwaySB.WinServiceHost.exe
wait $!
```

Скрипт вызывает сервис запуска сервисной шины и ожидает завершения ее работы.

Файл конфигурации сервиса
`/opt/flexberry-hwsb/NewPlatform.Flexberry.HighwaySB.WinServiceHost.exe.config`
содержит строку соединения с базой данных, расположенной по сетевому имени`FlexberryHWSBPostgres`:
```
Host=FlexberryHWSBPostgres;Port=5432;Database=flexberryhwsb;User ID=flexberry_orm_tester;Password=sa3dfE;
```

Для изменения параметров конфигурации при запуске контейнера можно заместить данный файл конфигурации файлом конфигурации, расположенном на HOST-системе.

На текущий момент docker-образ `dh.ics.perm.ru/flexberry/hwsb` (`dh.ics.perm.ru/flexberry/hwsb:latest`) является алиасом образа версии 2.0.0 `dh.ics.perm.ru/flexberry/hwsb:2.0.0`. В дальнейшем возможно появление образов следующих версий. Если Вам необходимо запустить образ определенной версии (например версию 2.0.0) необходимо при вызове указывать полное имя образа (`dh.ics.perm.ru/flexberry/hwsb:2.0.0`).

### Docker-образ редактора сервисной шины `dh.ics.perm.ru/flexberry/servicebuseditor`

Docker-образ `dh.ics.perm.ru/flexberry/servicebuseditor` построен на основе того же образа `dh.ics.perm.ru/kaf/alt.p8-mono4` с добавлением  файла `/etc/httpd2/conf/sites-available/vhosts.conf` конфигурации виртуального apache-хоста и 
каталога `/var/www/vhosts/ServiceBusEditor/`  содержащего код редактора.

Файл конфигурации Apache-сервера имеет вид:
```
Listen 80
NameVirtualHost *:80
<VirtualHost *:80>
  ServerName ServiceBusEditor.mono.ics.perm.ru
  ServerAdmin admin@server
  MonoServerPath ServiceBusEditor.mono.ics.perm.ru "/usr/bin/mod-mono-server4"
  MonoDebug ServiceBusEditor.mono.ics.perm.ru true
  MonoSetEnv ServiceBusEditor.mono.ics.perm.ru MONO_IOMAP=all
  MonoApplications ServiceBusEditor.mono.ics.perm.ru "/:/var/www/vhosts/ServiceBusEditor"
  AddDefaultCharset utf-8
  <Location "/">
    Allow from all
    Order allow,deny
    MonoSetServerAlias ServiceBusEditor.mono.ics.perm.ru
    SetHandler mono
    #SetOutputFilter DEFLATE
  </Location>
  ErrorLog /var/log/httpd2/ServiceBusEditor_error_log
  LogLevel debug
  CustomLog /var/log/httpd2/ServiceBusEditor_access_log common
</VirtualHost>
```

Файл конфигурации сервиса `/var/www/vhosts/ServiceBusEditor/Web.config` содержит строку соединения с базой данных, расположенной в домене `FlexberryHWSBPostgres`:
```
Host=FlexberryHWSBPostgres;Port=5432;Database=flexberryhwsb;User ID=flexberry_orm_tester;Password=sa3dfE;
```

Для изменения параметров конфигурации при запуске контейнера можно заместить данный файл конфигурации файлом конфигурации, расположенном на HOST-системе.

На текущий момент docker-образ `dh.ics.perm.ru/flexberry/servicebuseditor` (`dh.ics.perm.ru/flexberry/servicebuseditor:latest`) является алиасом образа версии 2.0.0 `dh.ics.perm.ru/flexberry/servicebuseditor:2.0.0`. В дальнейшем возможно появление образов следующих версий. Если Вам необходимо запустить образ определенной версии (например версию 2.0.0) необходимо при вызове указывать полное имя образа (`dh.ics.perm.ru/flexberry/servicebuseditor:2.0.0`).

### Docker-образ базы данных `dh.ics.perm.ru/kaf/alt.p8-postgresql9.6-ru`

Для работы с базой данных используется образ  `dh.ics.perm.ru/kaf/alt.p8-postgresql9.6-ru` с именованным томом базы  `postgresql-hwsb`.
Начальная (пустая) база данных `flexberryhwsb` содержится в файл-архиве `postgresql-hwsb.tgz`.
На его основе с помощью нижеприведенных команд формируется именованный том  `postgresql-hwsb`:
```
# docker volume create postgresql-hwsb
# tar -C /var/lib/docker/volumes/postgresql-hwsb/_data -xvzf postgresql-hwsb.tgz
```

При запуске именованный том `postgresql-hwsb` монтируется на каталог `/var/lib/pgsql/data/` контейнера.

### Запуск стека сервисов  сервисной шины `Highway Service Bus`

#### Запуск сервисов в стандартном `noswarm` режиме

##### Запуск сервиса `FlexberryHWSBPostgres`

Для запуска сервиса необходимо определить конфигурационный файлы `hg_hba.conf` и `postgresql.conf`  в каталоге 
`/etc/icsDockerCluster/confs/noswarm/postgresql-hwsb/conf/`.

В файле `hb_hba.conf`  должна присутствовать строка:
```
host    all             all             x.x.x.x/nn           md5
```
Где  `x.x.x.x/nn` — адрес локальной сети и сетевая маска, в которой располагаются запускаемые контейнеры.

Файл  `postgresql.conf` должен содержать строку
```
listen_addresses = '*'
```
обеспечивающий прослушивание запросов по всем сетевым интерфейсам.

Строка запуска контейнера выглядит следующим образом:
```
docker run -d \
        --name FlexberryHWSBPostgres \
        -v postgresql-hwsb:/var/lib/pgsql/data/ \
        -v /etc/icsDockerCluster/confs/noswarm/postgresql-hwsb/conf/:/conf \
        -p 5432:5432 \
        dh.ics.perm.ru/kaf/alt.p8-postgresql9.6-ru
```
Параметр
```
 -v postgresql-hwsb:/var/lib/pgsql/data/
```
обеспечивает монтирование каталога `/var/lib/pgsql/data/` контейнера на именованный том  `postgresql-hwsb` (см. выше).

Параметр
```
  -v /etc/icsDockerCluster/confs/noswarm/postgresql-hwsb/conf/:/conf
```
обеспечивает монтирование каталога файлов `hg_hba.conf`, `postgresql.conf` конфигурации postgres на каталог `/conf/` контейнера.

Параметр
```
  -p 5432:5432
```
обеспечивает проброс порта `5432` HOST-системы на порт `5432` контейнера.

Если порт  `5432` HOST-системы уже занят, необходимо указать другой свободный порт и описать этот порт в файлах конфигурации:
* `NewPlatform.Flexberry.HighwaySB.WinServiceHost.exe.config` контейнера сервисной шины
* `Web.config` редактора сервисной шины.

##### Запуск сервиса HWSB

Запуск сервиса осуществляется командой:
```
# docker run -d
      --name HWSB \
      -p 7075:7075 \
      -p 7085:7085 \
      --add-host FlexberryHWSBPostgres:x.x.x.x \
  dh.ics.perm.ru/flexberry/hwsb
```
Где  `x.x.x.x` — адрес хоста, где запущен сервис `FlexberryHWSBPostgres`.

Если файл конфигурации`NewPlatform.Flexberry.HighwaySB.WinServiceHost.exe.config`
отличается от стандартного, его необходимо разместить в каталоге `/etc/icsDockerCluster/confs/noswarm/hwsb/` и перекрыть параметром:
```
- v /etc/icsDockerCluster/confs/noswarm/hwsb/NewPlatform.Flexberry.HighwaySB.WinServiceHost.exe.config:/opt/flexberry-hwsb/NewPlatform.Flexberry.HighwaySB.WinServiceHost.exe.config
```
Параметры 
```
      -p 7075:7075 \
      -p 7085:7085 \
```
привязывают внешние порты `7075`, `7085` к соответствующим внутренним портам контейнера. В случае, если указанные внешние порты заняты  можно указать другие свободные порты `HOST-системы`.

Параметр 
```
      --add-host FlexberryHWSBPostgres:x.x.x.x \
```
(где x.x.x.x - адрес узла на котором запущен контейнер базы данных `FlexberryHWSBPostgres`) добавляет в файл `/etc/host`
запускаемого контейнера `HWSB` строку:
```
x.x.x.x FlexberryHWSBPostgres
```
которая обечпечивает трасляцию сетевого имени `FlexberryHWSBPostgres` в IP-адрес `x.x.x.x` при обращении ПО контейнера по сетевому имени (см. параметр `Host=FlexberryHWSBPostgres` файла конфигурации `NewPlatform.Flexberry.HighwaySB.WinServiceHost.exe.config`). 

##### Запуск сервиса `ServiceBusEditor`

Запуск сервиса осуществляется командой:
```
# docker run -d \
       --name ServiceBusEditor \
        -p 7080:80 \
        --add-host FlexberryHWSBPostgres:x.x.x.x \
        dh.ics.perm.ru/flexberry/servicebuseditor
```
Где  `x.x.x.x` — адрес хоста, где запущен сервис `FlexberryHWSBPostgres`.

Если файл конфигурации `Web.config` отличается от стандартного, его необходимо разместить в каталоге `/etc/icsDockerCluster/confs/noswarm/hwsb/` и перекрать параметром:
```
-v /etc/icsDockerCluster/confs/noswarm/servicebuseditor/Web.config:/var/www/vhosts/ServiceBusEditor/Web.config
```
Параметр
```
      -p 7080:80 \
```
привязывает внешний порт `7080` к внутреннему порту `80` контейнера. В случае, если внешний порт `7080` занят  можно указать другие свободныё порт `HOST-системы`.

Параметр 
```
      --add-host FlexberryHWSBPostgres:x.x.x.x \
```
(где x.x.x.x - адрес узла на котором запущен контейнер базы данных `FlexberryHWSBPostgres`) добавляет в файл `/etc/host`
запускаемого контейнера `ServiceBusEditor` строку:
```
x.x.x.x FlexberryHWSBPostgres
```
которая обечпечивает трасляцию сетевого имени `FlexberryHWSBPostgres` в IP-адрес `x.x.x.x` при обращении ПО контейнера по сетевому имени (см. параметр `Host=FlexberryHWSBPostgres` файла конфигурации `Web.config`). 

##### Пример запуска всех сервисов

Ниже приведен пример запуска всех сервисов на одном узле `10.130.5.94` с автоматическим формированием именовано тома `postgresql-hwsb`:
```
#!/bin/sh
volume=postgresql-hwsb
until docker volume inspect $volume >/dev/null 2>&1
do
        docker volume create $volume
        tar -C /var/lib/docker/volumes/$volume/_data -xvzf $volume.tgz
done

docker run -d \
        --name FlexberryHWSBPostgres \
        -v $volume:/var/lib/pgsql/data/ \
        -v /etc/icsDockerCluster/confs/noswarm/postgresql-hwsb/conf/:/conf \
        -p 5432:5432 \
        dh.ics.perm.ru/kaf/alt.p8-postgresql9.6-ru

docker run -d \
        --name HWSB \
        -p 7075:7075 \
        -p 7085:7085 \
        --add-host FlexberryHWSBPostgres:10.130.5.94 \
        -v /etc/icsDockerCluster/confs/noswarm/hwsb/NewPlatform.Flexberry.HighwaySB.WinServiceHost.exe.config:/opt/flexberry-hwsb/NewPlatform.Flexberry.HighwaySB.WinServiceHost.exe.config \
        dh.ics.perm.ru/flexberry/hwsb

docker run -d \
        --name ServiceBusEditor \
        -p 7080:80 \
        --add-host FlexberryHWSBPostgres:10.130.5.94 \
        -v /etc/icsDockerCluster/confs/noswarm/servicebuseditor/Web.config:/var/www/vhosts/ServiceBusEditor/Web.config \
        dh.ics.perm.ru/flexberry/servicebuseditor
```

Возможен запуск сервисов   `HWSB`, `ServiceBusEditor` на других узлах кластера. В этом случае удаленный запуск их можно осуществить командой `ssh`. Например:
```
# ssh  10.130.5.96  docker run --name HWSB ...
```
или путем удаленного входа на другой узел кластера 
```
# ssh 10.130.5.96
```
и запуска контейнера на этом узле
```
# docker run --name HWSB ...
```

#### Запуск сервисов в `swarm-кластере`

Запуск вышеописанных образов как сервисов в режиме `docker swarm` позволяет поддержать новые возможности:

* автоматический рестарт сервиса при его падении или обновлении образов на множестве допустимых для данного сервиса узлов;
* поддержка внутренней сети для взаимодествия сервисов `FlexberryHWSBPostgres`, `HWSB`, `ServiceBusEditor`;
* доступность сервисов `HWSB`, `ServiceBusEditor` на сетевых интерфейсах всех узлов кластера (`network mesh`);
* автоматическая поддержка доменных имен сервисов во внутренней сети;
* возможность масштабирования сервиса `ServiceBusEditor` на одном или нескольких узлов кластера;
* автоматическая балансировака нагрузки на масштабированный сервис `ServiceBusEditor`.

Режим `docker swarm` поддерживается в версиях docker `>=1.12` (август 2016 года).
До версии `1.13` (январь 2017-го) запуск сервисов возможен был только путем вызова команды:
```
# docker service create [OPTIONS] IMAGE [COMMAND] [ARG...] 
```
на manager-узле кластера.

Версии `1.13` и старше docker поддерживают режим запуска множества (`stack`) связанных сервисов, описанных в файле конфигурации `composeFileName` командой:
```
# docker stack deploy --compose-file composeFileName stackName
```

##### Создание внутренней overlay-сети HWSBNet

Для взаимодействия сервисов необходимо создать `overlay`-сеть. Данная сеть обеспечивает изолирование трафика между сервисами сети от внешего доступа и поддерживает внутренний `DNS-сервис`, обеспечивающий сетевую трансляцию имени сервиса в один или несколько внутренних сетевых адресов на которых функционирует сервис.

Создание сети осуществляется командой:
```
docker network create \
        --driver overlay \
        --subnet 192.168.1.0/27 \
        HWSBNet
```
Данная команда создает сеть `HWSBNet` для связывание сервисов в кластере (тип ) с подсетью на диапазоне адресов 192.168.1.0 - 192.168.1.31.

##### Независимый запуск с сервисов через `docker service create`

###### Запуск сервиса FlexberryHWSBPostgres

Для запуска сервиса необходимо определить конфигурационный файлы `hg_hba.conf` и `postgresql.conf`  в каталоге 
`/etc/icsDockerCluster/confs/swarm/postgresql-hwsb/conf/`.

В файле `hb_hba.conf`  должна присутствовать строка:
```
host    all             all             192.168.1.0/27           md5
```
Где  `192.168.1.0/27` — адрес локальной внутренней сети `HWSBNet` и сетевая маска, в которой располагаются запускаемые контейнеры.

Файл  `postgresql.conf` должен содержать строку
```
listen_addresses = '*'
```
обеспечивающий прослушивание запросов по всем сетевым интерфейсам.

Строка запуска контейнера выглядит следующим образом:
```
docker service create \
        --name FlexberryHWSBPostgres \
        --network HWSBNet \
        --mount type=volume,source=postgresql-hwsb,destination=/var/lib/pgsql/data/ \
        --mount type=bind,source=/etc/icsDockerCluster/confs/swarm/postgresql-hwsb/conf/,destination=/conf \
        --constraint "node.hostname == pm_db.node" \
        dh.ics.perm.ru/kaf/alt.p8-postgresql9.6-ru
```
Параметр:
```
        --network HWSBNet \
```
подключает запускаемый сервис к внутренней сети `HWSBNet`.

Параметр
```
 -mount type=volume,source=postgresql-hwsb,destination=/var/lib/pgsql/data/
```
обеспечивает монтирование каталога `/var/lib/pgsql/data/` контейнера на именованный том  `postgresql-hwsb` (см. выше).

Параметр
```
  --mount type=bind,source=/etc/icsDockerCluster/confs/swarm/postgresql-hwsb/conf/,destination=/conf
```
обеспечивает монтирование каталога файлов `hg_hba.conf`, `postgresql.conf` конфигурации postgres на каталог `/conf/` контейнера.

Обратите внимание, так как в доступе к базе данных данного сервиса из внешней сети нет необходимости в пробросе порта 5432 
во внешнюю сеть. Доступ к этому порту возможен только из сервисов подключенных к сети `HWSBNet`.
При запуске сервиса внутренний DNS сервер сети `HWSBNet` привязывает имя сервиса `FlexberryHWSBPostgres` к IP-адресу сервиса базы данных. Так что при запуске сервисов `HWSP`, `ServiceBusEditor` нет необходимости править файл `/etc/hosts` сервиса, так как трансляцию имени сервиса `FlexberryHWSBPostgres` в текущий сетевой адрес сервера базы данных в внутренней сети `HWSBNet`.

Параметр
```
        --constraint "node.hostname == pm_db.node" \
```
ограничивает множество узлов, на которых возможен запуск сервиса `FlexberryHWSBPostgres`. В данном случае запуск сервиса возможен только на узле `pm_db.node` (`10.130.5.94`). 


###### Запуск сервиса HWSB



###### Запуск сервиса ServiceBusEditor




##### Запуск в режиме стека `docker stack`





