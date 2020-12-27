# Лабораторная работа 6

## Задание

1) Создать свой RPM пакет (можно взять свое приложение, либо собрать, например, nginx с определенными опциями);
2) Создать свой репозиторий и разместить там ранее собранный RPM.

## Процесс

Скачаем нужные пакеты и установим их

```bash
curl -OL https://nginx.org/packages/centos/7/SRPMS/nginx-1.18.0-2.el7.ngx.src.rpm
rpm -i nginx-1.18.0-2.el7.ngx.src.rpm
curl -OL https://www.openssl.org/source/latest.tar.gz
tar -xvf latest.tar.gz
sudo yum-builddep -y rpmbuild/SPECS/nginx.spec
```

Редактируем конфигурационный файл пакета, чтобы собрать его же вместе с `openssl`

```bash
$ sudo yum install -y rpm-build perl gcc nano make
$ diff -u --color=always _nginx.spec rpmbuild/SPECS/nginx.spec | less -R
$ nano rpmbuild/SPECS/nginx.spec
# Добавим в %build: --with-openssl=/root/openssl-1.1.1i \
```

Соберём пакет

```bash
sudo rpmbuild -bb rpmbuild/SPECS/nginx.spec
```

Собираться будет долго. Ждем...

Теперь осталось убедиться, что пакеты создались

```bash
$ sudo ls /root/rpmbuild/RPMS/x86_64/
nginx-1.18.0-2.el8.ngx.x86_64.rpm  nginx-debuginfo-1.18.0-2.el8.ngx.x86_64.rpm
```

Установим nginx из локального репозитория и запустим его

```bash
$ sudo yum localinstall -y /root/rpmbuild/RPMS/x86_64/nginx-1.18.0-2.el7.ngx.x86_64.rpm
...
$ sudo systemctl start nginx
$ sudo systemctl status nginx
● nginx.service - nginx - high performance web server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: active (running) since Sun 2020-12-27 17:15:17 UTC; 3s ago
     Docs: http://nginx.org/en/docs/
  Process: 62234 ExecStart=/usr/sbin/nginx -c /etc/nginx/nginx.conf (code=exited, status=0/SUCCESS)
 Main PID: 62235 (nginx)
    Tasks: 2 (limit: 5972)
   Memory: 2.0M
   CGroup: /system.slice/nginx.service
           ├─62235 nginx: master process /usr/sbin/nginx -c /etc/nginx/nginx.conf
           └─62236 nginx: worker process

Dec 27 17:15:17 localhost.localdomain systemd[1]: Starting nginx - high performance web server...
Dec 27 17:15:17 localhost.localdomain systemd[1]: nginx.service: Can't open PID file /var/run/nginx.pid (yet?) after start: No such file or directory
Dec 27 17:15:17 localhost.localdomain systemd[1]: Started nginx - high performance web server.
```

Создадим каталог repo в директории nginx, скопируем туда наш собранный RPM

```bash
sudo mkdir /usr/share/nginx/html/repo
sudo cp /root/rpmbuild/RPMS/x86_64/nginx-1.18.0-2.el7.ngx.x86_64.rpm /usr/share/nginx/html/repo/
```

Инициализируем репозиторий

```bash
$ sudo yum install -y createrepo
...
$ sudo createrepo /usr/share/nginx/html/repo/
Directory walk started
Directory walk done - 1 packages
Temporary output repo path: /usr/share/nginx/html/repo/.repodata/
Preparing sqlite DBs
Pool started (with 5 workers)
Pool finished
```

Добавим `autoindex on;` в конфиг фаил nginx Проверим конфиг и перезапустим nginx

```bash
$ sudo nginx -t
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
$ sudo systemctl reload nginx
```

Проверим работу репозитория

```bash
$ curl -a http://localhost/repo/
<html>
<head><title>Index of /repo/</title></head>
<body>
<h1>Index of /repo/</h1><hr><pre><a href="../">../</a>
<a href="repodata/">repodata/</a>                                          27-Dec-2020 17:20                   -
<a href="nginx-1.18.0-2.el8.ngx.x86_64.rpm">nginx-1.18.0-2.el8.ngx.x86_64.rpm</a>                  27-Dec-2020 17:20             2222324
</pre><hr></body>
</html>
```

Добавим репозиторий в yum

```bash
$ sudo tee /etc/yum.repos.d/mai.repo << EOF
[andrey]
name=andrey-linux
baseurl=http://localhost/repo
gpgcheck=0
enabled=1
EOF
```

И посмотрим, ответит ли он нам

```bash
$ yum repolist enabled | grep andrey
andrey                         andrey-linux
```

Установим nginx из локального репозитория

```bash
$ sudo yum reinstall nginx
Loaded plugins: fastestmirror
Loading mirror speeds from cached hostfile
 * base: mirror.logol.ru
 * extras: mirror.logol.ru
 * updates: mirror.reconn.ru
Resolving Dependencies
--> Running transaction check
---> Package nginx.x86_64 1:1.18.0-2.el7.ngx will be reinstalled
--> Finished Dependency Resolution

Dependencies Resolved

==============================================================================================================================================================================================
 Package                                   Arch                                       Version                                                   Repository                               Size
==============================================================================================================================================================================================
Reinstalling:
 nginx                                     x86_64                                     1:1.18.0-2.el7.ngx                                        andrey                                   2.1 M

Transaction Summary
==============================================================================================================================================================================================
Reinstall  1 Package

Total download size: 2.1 M
Installed size: 6.0 M
Is this ok [y/d/N]: y
Downloading packages:
nginx-1.18.0-2.el7.ngx.x86_64.rpm                                                                                                                                      | 2.1 MB  00:00:00     
Running transaction check
Running transaction test
Transaction test succeeded
Running transaction
  Installing : 1:nginx-1.18.0-2.el7.ngx.x86_64                                                                                                                                            1/1 
  Verifying  : 1:nginx-1.18.0-2.el7.ngx.x86_64                                                                                                                                            1/1 

Installed:
  nginx.x86_64 1:1.18.0-2.el7.ngx                                                                                                                                                             

Complete!
```

Проверим список пакетов

```bash
$ yum list | grep andrey
nginx.x86_64                                1:1.18.0-2.el7.ngx         @andrey     
```

Урааа! Репозиторий работает
