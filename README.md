# docker

## Docker-образы сервисной шины Highway Service Bus


Стек docker-сервисов сервисной шины Highway Service Bus включает в себя два docker-образа:
* dh.ics.perm.ru/flexberry/hwsb — docker-образ сервисной шины Highway Service Bus;
* dh.ics.perm.ru/flexberry/servicebuseditor -   docker-образ редактора сервисной шины.

Кроме этого для работы с базой данных сервисной шины используется образ `dh.ics.perm.ru/kaf/alt.p8-postgresql9.6-ru` с именованным томом `postgresql-hwsb`,содержащим postgres базу данных `FlexberryHWSB` с пользователем `flexberry_orm_tester`.

### Docker-образ сервисной шины dh.ics.perm.ru/flexberry/hwsb

Docker-образ `dh.ics.perm.ru/flexberry/hwsb` построен на основе образа dh.ics.perm.ru/kaf/alt.p8-mono4 с добавлением пользователя highway (uid:504, gid:504) и каталога flexberry-hwsb, содержащего код сервисной шины. Каталог  flexberry-hwsb располагается в каталоге /opt/ файловой системы образа.

При запуске контейнера вызывается shell-скрипт /opt/startFlexberry-hwsb.sh:
```
#!/bin/sh
. /usr/bin/mono-service2 -l:/tmp/highwaysb.lock -d:/opt/flexberry-hwsb -m:highwaysb NewPlatform.Flexberry.HighwaySB.WinServiceHost.exe
wait $!
```

Скрипт вызывает сервис запуска сервисной шины и ожидает завершения ее работы.

Файл конфигурации сервиса
/opt/flexberry-hwsb/NewPlatform.Flexberry.HighwaySB.WinServiceHost.exe.config
содержит строку соединения с базой данных, расположенной в домене FlexberryHWSBPostgres:
```
Host=FlexberryHWSBPostgres;Port=5432;Database=flexberryhwsb;User ID=flexberry_orm_tester;Password=sa3dfE;
```

Для изменения параметров конфигурации при запуске контейнера можно заместить данный файл конфигурации файлом конфигурации, расположенном на HOST-системе.

### Docker-образ редактора сервисной шины dh.ics.perm.ru/flexberry/servicebuseditor

Docker-образ dh.ics.perm.ru/flexberry/hwsb построен на основе того же образа dh.ics.perm.ru/kaf/alt.p8-mono4 с добавлением  файла /etc/httpd2/conf/sites-available/vhosts.conf конфигурации виртуального apache-хоста и 
каталога /var/www/vhosts/ServiceBusEditor/  содержащего код редактора.

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

Файл конфигурации сервиса /var/www/vhosts/ServiceBusEditor/Web.config содержит строку соединения с базой данных, расположенной в домене FlexberryHWSBPostgres:
```
Host=FlexberryHWSBPostgres;Port=5432;Database=flexberryhwsb;User ID=flexberry_orm_tester;Password=sa3dfE;
```

Для изменения параметров конфигурации при запуске контейнера можно заместить данный файл конфигурации файлом конфигурации, расположенном на HOST-системе.

### Docker-образ базы данных

Для работы с базой данных используется образ  dh.ics.perm.ru/kaf/alt.p8-postgresql9.6-ru с именованным томом базы  postgresql-hwsb.
Начальная (пустая) база данных flexberryhwsb содержится в файл-архиве postgresql-hwsb.tgz.
На его основе с помощью нижеприведенных команд формируется именованный том  postgresql-hwsb:
```
# docker volume create postgresql-hwsb
# tar -C /var/lib/docker/volumes/postgresql-hwsb/_data -xvzf postgresql-hwsb.tgz
```

При запуске именованный том postgresql-hwsb монтируется на каталог /var/lib/pgsql/data/ контейнера.

### Запуск стека сервисов  сервисной шины Highway Service Bus

#### Запуск сервисов в стандартном noswarm режиме

##### Запуск сервиса FlexberryHWSBPostgres

Для запуска сервиса необходимо определить конфигурационный файлы hg_hba.conf и postgresql.conf  в каталоге 
/etc/icsDockerCluster/confs/noswarm/postgresql-hwsb/conf/.

В файле hb_hba.conf  должна присутствовать строка:
```
host    all             all             x.x.x.x/nn           md5
```
Где  x.x.x.x/nn — адрес локальной сети и сетевая маска, в которой располагаются запускаемые контейнеры.

Файл  postgresql.conf должен содержать строку
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
обеспечивает монтирование каталога var/lib/pgsql/data/ контейнера на именованный том  postgresql-hwsb (см. выше).

Параметр
```
  -v /etc/icsDockerCluster/confs/noswarm/postgresql-hwsb/conf/:/conf
```
обеспечивает монтирование каталога файлов hg_hba.conf, postgresql.conf конфигурации postgres на каталог /conf контейнера.

Параметр
```
  -p 5432:5432
```
обеспечивает проброс порта 5432 HOST-системы на порт 5432 контейнера.

Если порт  5432 HOST-системы уже занят, необходимо указать другой свободный порт и описать этот порт в файлах конфигурации:
* NewPlatform.Flexberry.HighwaySB.WinServiceHost.exe.config контейнера сервисной шины
* Web.config редактора сервисной шины.

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
Где  x.x.x.x — адрес хоста, где запущен сервис FlexberryHWSBPostgres.

Если файл конфигурации NewPlatform.Flexberry.HighwaySB.WinServiceHost.exe.config
отличается от стандартного, его необходимо разместить в каталоге /etc/icsDockerCluster/confs/noswarm/hwsb/ и перекрыть параметром:
```
- v /etc/icsDockerCluster/confs/noswarm/hwsb/NewPlatform.Flexberry.HighwaySB.WinServiceHost.exe.config:/opt/flexberry-hwsb/NewPlatform.Flexberry.HighwaySB.WinServiceHost.exe.config
```

##### Запуск сервиса ServiceBusEditor

Запуск сервиса осуществляется командой:
```
# docker run -d \
       --name ServiceBusEditor \
        -p 7080:80 \
        --add-host FlexberryHWSBPostgres:x.x.x.x \
        dh.ics.perm.ru/flexberry/servicebuseditor
```
Где  x.x.x.x — адрес хоста, где запущен сервис FlexberryHWSBPostgres.

Если файл конфигурации Web.config отличается от стандартного, его необходимо разместить в каталоге /etc/icsDockerCluster/confs/noswarm/hwsb/ и перекрать параметром:
```
-v /etc/icsDockerCluster/confs/noswarm/servicebuseditor/Web.config:/var/www/vhosts/ServiceBusEditor/Web.config
```

##### Пример запуска всех сервисов

Ниже приведен пример запуска всех сервисов на одном узле 10.130.5.94 с автоматическим формированием именовано тома postgresql-hwsb:
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

Возможен запуск сервисов   HWSB, ServiceBusEditor на других узлах кластера. В этом случае запуск их можно осуществить командой ssh. Например:
```
# ssh  10.130.5.96  docker run --name HWSB ...
```

#### Запуск сервисов в swarm-кластере

##### Независимый запуск через shell-скрипт

##### Запуск в режиме стека docker stack



