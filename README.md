# DZ08SystemD
> Написать service, который будет раз в 30 секунд мониторить лог на предмет наличия ключевого слова (файл лога и ключевое слово должны задаваться в /etc/default).  
> Установить spawn-fcgi и создать unit-файл (spawn-fcgi.sevice) с помощью переделки init-скрипта (https://gist.github.com/cea2k/1318020).  
> Доработать unit-файл Nginx (nginx.service) для запуска нескольких инстансов сервера с разными конфигурационными файлами одновременно.  
```
gor@testsrv:~$ sudo tee /etc/default/watchlog > /dev/null << 'EOF'
> WORD="ALERT"
> LOG="/var/log/watchlog.log"
> EOF
gor@testsrv:~$ sudo touch /var/log/watchlog.log
gor@testsrv:~$ sudo tee /opt/watchlog.sh > /dev/null <<'EOF'
> #!/bin/bash
> WORD="$1"
> LOG="$2"
> DATE=$(date +"%a %b %d %H:%M:%S %Z %Y")
> if grep -q "$WORD" "$LOG"; then
>  logger "$DATE: I found word, Master!"
> fi
> EOF
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
```
>вышеприведенный скрипт почему-то не работал
```
#!/bin/bash
#WORD="$1"
#LOG="$2"
echo "Debug: WORD=$WORD, LOG=$LOG" >> /tmp/debug.log
DATE=$(date +"%a %b %d %H:%M:%S %Z %Y")
if grep "$WORD" "$LOG"; then
 logger "$DATE: I found word, Master!"
 echo "Found: $WORD in $LOG" >> /tmp/debug.log
else
    echo "Not found: $WORD in $LOG" >> /tmp/debug.log
fi

gor@testsrv:~$ sudo tail -n 10 /var/log/syslog  | grep word
2025-12-03T12:07:51.227146+00:00 testsrv systemd[1]: Starting watchlog.service - Watchlog service — monitors log for keyword...
2025-12-03T12:07:51.245044+00:00 testsrv root: Ср дек 03 12:07:51 UTC 2025: I found word, Master!
2025-12-03T12:07:51.247946+00:00 testsrv systemd[1]: Finished watchlog.service - Watchlog service — monitors log for keyword.
2025-12-03T12:08:23.755623+00:00 testsrv systemd[1]: Starting watchlog.service - Watchlog service — monitors log for keyword...
2025-12-03T12:08:23.768012+00:00 testsrv root: Ср дек 03 12:08:23 UTC 2025: I found word, Master!
2025-12-03T12:08:23.771338+00:00 testsrv systemd[1]: Finished watchlog.service - Watchlog service — monitors log for keyword.
```
> а вот так заработало...
