Научится проектировать централизованный сбор логов.

Перезупустим службу NTP Chrony: '''systemctl restart chronyd'''

Проверим, что служба работает корректно:
[root@web ~]# systemctl status chronyd
   Loaded: loaded (/usr/lib/systemd/system/chronyd.service; enabled; vendor preset: enabled)
   Active: active (running) since Wed 2022-04-13 08:34:13 UTC; 15s ago
     Docs: man:chronyd(8)
           man:chrony.conf(5)

Установим текстовый редактор VIM на обоих серверах:
[root@web ~]# yum install -y vim
[root@log ~]# yum install -y vim


Установка nginx на виртуальной машине web
[root@web ~]# yum install epel-release

[root@web ~]# yum install nginx
Installed:
  nginx.x86_64 1:1.20.1-9.el7

Проверим статус nginx
[root@web ~]# systemctl start nginx
[root@web ~]# systemctl status nginx
 nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; enabled; vendor preset: enabled)
   Active: active (running) since Wed 2022-04-13 08:41:14 UTC; 7s ago
Main PID: 23767 (nginx)
   CGroup: /system.slice/nginx.service
           ├─23767 nginx: master process /usr/sbin/nginx
           └─23769 nginx: worker process

[root@web ~]# ss -tln | grep 80
LISTEN     0      128          *:80                       *:*
LISTEN     0      128       [::]:80                    [::]:*


Работу nginx можно проверить на хосте, в браузере введем в адерсную строку http://192.168.50.10
Страница открывается: Welcom to CentOS



Переходим на виртуальную машину log

Проверяем, установлен ли rsyslog:
[root@log ~]# yum list rsyslog
rsyslog.x86_64 8.24.0-57.el7_9.2 updates


В файле /etc/rsyslog.conf вносим изменения, открываем порт 514 (TCP и UDP)
[root@log ~]# vim /etc/rsyslog.conf

приводим к виду:
# provides UDP syslog reception
module(load="imudp")
input(type="imudp" port="514")

# provides TCP syslog reception
module(load="imtcp")
input(type="imtcp" port="514")

В конец файла добавляем правила приёма сообщений от хостов:
#Add remote logs
$template RemoteLogs,"/var/log/rsyslog/%HOSTNAME%/%PROGRAMNAME%.log"
*.* ?RemoteLogs
& ~
#Данные параметры будут отправлять в папку /var/log/rsyslog логи, которые будут приходить от
других серверов. Например, Access-логи nginx от сервера web, будут идти в файл
/var/log/rsyslog/web/nginx_access.log

Проверяем, после перезапуска службы rsyslog
[root@log ~]# systemctl restart rsyslog
[root@log ~]# ss -tuln
Netid  State      Recv-Q Send-Q            Local Address:Port                           Peer Address:Port
udp    UNCONN     0      0                             *:514                                       *:*
udp    UNCONN     0      0                     127.0.0.1:323                                       *:*
...
tcp    LISTEN     0      100                   127.0.0.1:25                                        *:*
tcp    LISTEN     0      25                            *:514                                       *:*
...
tcp    LISTEN     0      100                       [::1]:25                                     [::]:*
tcp    LISTEN     0      25                        [::]:514                                     [::]:*
  


Переходим на виртуальную машину web
Проверим версию nginx:
[root@web ~]# rpm -qa | grep nginx
nginx-1.20.1-9.el7.x86_64
nginx-filesystem-1.20.1-9.el7.noarch

Наша версия 1.20

Редактируем /etc/nginx/nginx.conf
[root@web ~]# vim /etc/nginx/nginx.conf
Приводим к виду:

error_log /var/log/nginx/error.log;
error_log syslog:server=192.168.50.15:514,tag=nginx_error;

access_log syslog:server=192.168.50.15:514,tag=nginx_access,severity=info combined;

Проверяем, что конфигурация nginx указана правильно:
[root@web ~]# nginx -t
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful

Удаляем картинку, к которой будет обращаться nginx во время открытия веб-сраницы (для проверки отправки логов):
[root@web ~]# rm /usr/share/nginx/html/img/header-background.png


Переходим на виртуальную машину log
Смотрим информацию об nginx:
[root@log web]# cat /var/log/rsyslog/web/nginx_access.log
[root@log web]# cat /var/log/rsyslog/web/nginx_access.log
#скрин логов в папке с ДЗ
