# Демонстрация изолированной программной среды средствами systemd

Покажем возможность изолировать программу средствами systemd, который в свою очередь использует различные срества ядра Linux.

Создайте файл `/usr/local/bin/systemd-isolation-demo` со следующим текстом:

```
#!/bin/sh -x
echo $UID
ping -c3 -W3 8.8.8.8
ping -c3 -W3 127.0.0.1
ls -la /var/tmp
echo 123 > /var/tmp/systemd-isolation-demo
cat /var/tmp/systemd-isolation-demo
ls -la /home
exit 0
```

Сделайте его исполняемым:  
`chmod +x /usr/local/bin/systemd-isolation-demo`

Создайте файл `/etc/systemd/system/systemd-isolation-demo.service` со следующим текстом:

```
[Unit]
Description=systemd isolation demo

[Service]
Type=oneshot
RemainAfterExit=yes
ExecStart=/usr/local/bin/systemd-isolation-demo

User=nobody
Group=nobody

[Install]
WantedBy=multi-user.target
```

Выполните `systemctl daemon-reload`.

Теперь запустите этот как службу systemd:  
`systemctl start systemd-isolation-demo`

Посмотрите `systemctl status systemd-isolation-demo` и `journalctl -b 0 -u systemd-isolation-demo | cat -`. Увидите, что все прописанные в скрипте команды были выполнены успешно.

Выполните `cat /var/tmp/systemd-isolation-demo` и убедитесь, что было выведено: `123`. Удалите этот файл: `rm -fv /var/tmp/systemd-isolation-demo`.

Таким образом, зафиксируем, что:
* удалось получить доступ к глобальной сети (`8.8.8.8`)
* удалось получить доступ к лоакальному петлевому интерфейсу (`127.0.0.1`)
* удалось получить доступ к директории `/var/tmp` и сделать ее листинг
* удалось записать в файл `/var/tmp/systemd-isolation-demo`, который затем удалось прочитать другому процессу — вашей консоли и ее дочерним процессам
* удалось получить доступ к директории `/home` и сделать ее листинг

Теперь приведите файл `/etc/systemd/system/systemd-isolation-demo.service` к следующему виду:

```
[Unit]
Description=systemd isolation demo

[Service]
Type=oneshot
RemainAfterExit=yes
ExecStart=/usr/local/bin/systemd-isolation-demo

User=nobody
Group=nobody

PrivateNetwork=yes
ProtectHome=yes
PrivateTmp=yes

[Install]
WantedBy=multi-user.target
```

Выполните `systemctl daemon-reload`. Выполните `systemctl restart systemd-isolation-demo`. Посмотрите лог работы скрипта: `journalctl -b 0 -u systemd-isolation-demo | cat -`.

Выполните `cat /var/tmp/systemd-isolation-demo` и убедитесь, что такой файл вам недоступен, хотя в логе работы скрипта видно, что он был создан.

Таким образом, зафиксируем, что:
* был заблокирован доступ к глобальной сети (`8.8.8.8`)
* доступ к локальному петлевому интерфейсу (`127.0.0.1`) был оставлен
* используется изолированная и доступная только этой службе systemd директория `/var/tmp`
* доступ к директории `/home` был заблокирован

Таким образом, проверили возможность создания изолированной программной среды с помощью systemd.

