# DZ08SystemD
> Написать service, который будет раз в 30 секунд мониторить лог на предмет наличия ключевого слова (файл лога и ключевое слово должны задаваться в /etc/default).  
> Установить spawn-fcgi и создать unit-файл (spawn-fcgi.sevice) с помощью переделки init-скрипта (https://gist.github.com/cea2k/1318020).  
> Доработать unit-файл Nginx (nginx.service) для запуска нескольких инстансов сервера с разными конфигурационными файлами одновременно.  
> 1
```
gor@testsrv:~$ sudo tee /etc/default/watchlog > /dev/null << 'EOF'
> WORD="ALERT"
> LOG="/var/log/watchlog.log"
> EOF
gor@testsrv:~$ sudo touch /var/log/watchlog.log
gor@testsrv:~$ sudo cat /opt/watchlog.sh
#!/bin/bash
#WORD="$1"
#LOG="$2"
#echo "Debug: WORD=$WORD, LOG=$LOG" >> /tmp/debug.log
DATE=$(date +"%a %b %d %H:%M:%S %Z %Y")
if grep "$WORD" "$LOG"; then
 logger "$DATE: I found word, Master!"
# echo "Found: $WORD in $LOG" >> /tmp/debug.log
#else
#    echo "Not found: $WORD in $LOG" >> /tmp/debug.log
fi

gor@testsrv:~$ sudo chmod +x /opt/watchlog.sh
gor@testsrv:~$ sudo tee /etc/systemd/system/watchlog.service > /dev/null <<'EOF'
> [Unit]
> Description=Watchlog service — monitors log for keyword
>
> [Service]
> Type=oneshot
> EnvironmentFile=/etc/default/watchlog
> ExecStart=/opt/watchlog.sh "$WORD" "$LOG"
> EOF
gor@testsrv:~$ sudo tee /etc/systemd/system/watchlog.timer > /dev/null <<'EOF'
> [Unit]
> Description=Run watchlog service every 30 seconds
>
> [Timer]
> OnUnitActiveSec=30
> Unit=watchlog.service
>
> [Install]
> WantedBy=multi-user.target
> EOF
gor@testsrv:~$ sudo systemctl daemon-reexec
gor@testsrv:~$ sudo systemctl start watchlog.timer
gor@testsrv:~$ sudo systemctl status watchlog.timer
● watchlog.timer - Run watchlog service every 30 seconds
     Loaded: loaded (/etc/systemd/system/watchlog.timer; disabled; preset: enabled)
     Active: active (elapsed) since Wed 2025-12-03 10:18:46 UTC; 7s ago
    Trigger: n/a
   Triggers: ● watchlog.service

дек 03 10:18:46 testsrv systemd[1]: Started watchlog.timer - Run watchlog service every 30 seconds.

gor@testsrv:~$ sudo tail -n 10 /var/log/syslog  | grep word
2025-12-03T12:07:51.227146+00:00 testsrv systemd[1]: Starting watchlog.service - Watchlog service — monitors log for keyword...
2025-12-03T12:07:51.245044+00:00 testsrv root: Ср дек 03 12:07:51 UTC 2025: I found word, Master!
2025-12-03T12:07:51.247946+00:00 testsrv systemd[1]: Finished watchlog.service - Watchlog service — monitors log for keyword.
2025-12-03T12:08:23.755623+00:00 testsrv systemd[1]: Starting watchlog.service - Watchlog service — monitors log for keyword...
2025-12-03T12:08:23.768012+00:00 testsrv root: Ср дек 03 12:08:23 UTC 2025: I found word, Master!
2025-12-03T12:08:23.771338+00:00 testsrv systemd[1]: Finished watchlog.service - Watchlog service — monitors log for keyword.
```
> 2
```
gor@testsrv:~$ sudo apt install spawn-fcgi php php-cgi php-cli \
>  apache2 libapache2-mod-fcgid -y
gor@testsrv2:~$ cat  /etc/systemd/system/spawn-fcgi.service
[Unit]
Description=Spawn-fcgi startup service by Otus
After=network.target

[Service]
Type=simple
PIDFile=/run/spawn-fcgi.pid
EnvironmentFile=/etc/spawn-fcgi/fcgi.conf
ExecStart=/usr/bin/spawn-fcgi -n $OPTIONS
KillMode=process

[Install]
WantedBy=multi-user.target

gor@testsrv2:~$ sudo systemctl daemon-reload
gor@testsrv2:~$ sudo systemctl start spawn-fcgi
gor@testsrv2:~$ sudo systemctl status spawn-fcgi.service
● spawn-fcgi.service - Spawn-fcgi startup service by Otus
     Loaded: loaded (/etc/systemd/system/spawn-fcgi.service; disabled; preset: enabled)
     Active: active (running) since Wed 2025-12-03 12:45:58 UTC; 13s ago
   Main PID: 26033 (php-cgi)
      Tasks: 33 (limit: 4595)
     Memory: 14.5M (peak: 14.8M)
        CPU: 41ms
     CGroup: /system.slice/spawn-fcgi.service
             ├─26033 /usr/bin/php-cgi
             ├─26034 /usr/bin/php-cgi
             ├─26035 /usr/bin/php-cgi
             ├─26036 /usr/bin/php-cgi
             ├─26037 /usr/bin/php-cgi
             ├─26038 /usr/bin/php-cgi
             ├─26039 /usr/bin/php-cgi
             ├─26040 /usr/bin/php-cgi
             ├─26041 /usr/bin/php-cgi

```
> исправлено PIDFile=/var/run/spawn-fcgi.pid на PIDFile=/run/spawn-fcgi.pid, иначе сервис не стартует  
> 3
```
gor@testsrv:~$ sudo cat /etc/systemd/system/nginx@.service
[Unit]
Description=A high performance web server and a reverse proxy server
Documentation=man:nginx(8)
After=network.target nss-lookup.target

[Service]
Type=forking
PIDFile=/run/nginx-%I.pid
ExecStartPre=/usr/sbin/nginx -t -c /etc/nginx/nginx-%I.conf -q -g 'daemon on; master_process on;'
ExecStart=/usr/sbin/nginx -c /etc/nginx/nginx-%I.conf -g 'daemon on; master_process on;'
ExecReload=/usr/sbin/nginx -c /etc/nginx/nginx-%I.conf -g 'daemon on; master_process on;' -s reload
ExecStop=-/sbin/start-stop-daemon --quiet --stop --retry QUIT/5 --pidfile /run/nginx-%I.pid
TimeoutStopSec=5
KillMode=mixed

[Install]
WantedBy=multi-user.target

gor@testsrv:~$ sudo touch /etc/nginx/nginx-first.conf
gor@testsrv:~$ sudo touch /etc/nginx/nginx-second.conf

gor@testsrv:~$ sudo cat /etc/nginx/nginx-first.conf
user www-data;
worker_processes auto;
#pid /run/nginx.pid;
pid /run/nginx-first.pid;
error_log /var/log/nginx/error.log;
include /etc/nginx/modules-enabled/*.conf;

events {
        worker_connections 768;
        # multi_accept on;
}

http {
        server {
                listen 9001;
        }
...

gor@testsrv:~$ sudo cat /etc/nginx/nginx-second.conf
user www-data;
worker_processes auto;
#pid /run/nginx.pid;
pid /run/nginx-second.pid;
error_log /var/log/nginx/error.log;
include /etc/nginx/modules-enabled/*.conf;

events {
        worker_connections 768;
        # multi_accept on;
}

http {
        server {
                listen 9002;
        }
...

gor@testsrv:~$ sudo ss -tnulp | grep nginx
tcp   LISTEN 0      511                 0.0.0.0:9002       0.0.0.0:*    users:(("nginx",pid=27708,fd=5),("nginx",pid=27707,fd=5))
tcp   LISTEN 0      511                 0.0.0.0:9001       0.0.0.0:*    users:(("nginx",pid=27592,fd=5),("nginx",pid=27591,fd=5))
tcp   LISTEN 0      511                 0.0.0.0:80         0.0.0.0:*    users:(("nginx",pid=1880,fd=5),("nginx",pid=1876,fd=5))
tcp   LISTEN 0      511                    [::]:80            [::]:*    users:(("nginx",pid=1880,fd=6),("nginx",pid=1876,fd=6))

gor@testsrv:~$ sudo ps afx | grep nginx
   1876 ?        Ss     0:00 nginx: master process /usr/sbin/nginx -g daemon on; master_process on;
   1880 ?        S      0:00  \_ nginx: worker process
  27767 pts/1    S+     0:00              \_ grep --color=auto nginx
  27591 ?        Ss     0:00 nginx: master process /usr/sbin/nginx -c /etc/nginx/nginx-first.conf -g daemon on; master_process on;
  27592 ?        S      0:00  \_ nginx: worker process
  27707 ?        Ss     0:00 nginx: master process /usr/sbin/nginx -c /etc/nginx/nginx-second.conf -g daemon on; master_process on;
  27708 ?        S      0:00  \_ nginx: worker process
```




