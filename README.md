# Домашнее задание
Что нужно сделать?

1. Запустить nginx на нестандартном порту 3-мя разными способами:
+ переключатели setsebool;
+ добавление нестандартного порта в имеющийся тип;
+ формирование и установка модуля SELinux.
К сдаче:
+ README с описанием каждого решения (скриншоты и демонстрация приветствуются).
2. Обеспечить работоспособность приложения при включенном selinux.
+ развернуть приложенный стенд https://github.com/mbfx/otus-linux-adm/tree/master/selinux_dns_problems;
+ выяснить причину неработоспособности механизма обновления зоны (см. README);
+ предложить решение (или решения) для данной проблемы;
+ выбрать одно из решений для реализации, предварительно обосновав выбор;
+ реализовать выбранное решение и продемонстрировать его работоспособность.
К сдаче:
+ README с анализом причины неработоспособности, возможными способами решения и обоснованием выбора одного из них;
+ исправленный стенд или демонстрация работоспособной системы скриншотами и описанием.
В чат ДЗ отправьте ссылку на ваш git-репозиторий. Обычно мы проверяем ДЗ в течение 48 часов.
Если возникнут вопросы, обращайтесь к студентам, преподавателям и наставникам в канал группы в Slack.
Удачи при выполнении!

# SELinux: Запустить nginx на нестандартном порту 3-мя разными способами
### Способ 1 
#### Разрешим в SELinux работу nginx на порту TCP 4881 c помощью переключателей setsebool
Проверим, что в ОС отключен файервол **systemctl status firewalld**

![Image alt](/image/selinux1.png)

Проверяем, что  конфигурация nginx  настроена без ошибок: **nginx -t**

![Image alt](/image/selinux2.png)

Проверяем режим работы SELinux: **getenforce**

![Image alt](/image/selinux3.png)

Режим **Enforcing** означает, что SELinux будет блокировать запрещенную активность
__Разрешим в SELinux работу nginx на порту TCP 4881 c помощью переключателей setsebool__
Находим в логах (/var/log/audit/audit.log) информацию о блокировании порта

![Image alt](/image/selinux4.png)

В ОС отсутствует **audit2why**, его необходимо установить, устанавливаем  пакет **policycoreutils-python**
```
yum install policycoreutils-python
```

Копируем время, в которое был записан этот лог, и, с помощью утилиты audit2why смотрим информации о запрете: **grep 1676738964.624:1491 /var/log/audit/audit.log | audit2why**

![Image alt](/image/selinux5.png)

Из вывода утилиты **audit2why**  необходимо поменять параметр **nis_enabled**
Включаем параметр nis_enabled и перезапустим nginx: **setsebool -P nis_enabled on**

![Image alt](/image/selinux6.png)

Проверим статус параметра **nis_enabled**
```
[root@selinux vagrant]# getsebool -a | grep nis_enabled
nis_enabled --> on
```
### Способ 2
#### Теперь разрешим в SELinux работу nginx на порту TCP 4881 c помощью добавления нестандартного порта в имеющийся тип:
Поиск имеющегося типа, для http трафика: **semanage port -l | grep http**

![Image alt](/image/selinux7.png)

Добавим порт в тип http_port_t: **semanage port -a -t http_port_t -p tcp 4881** , проверяем порт **semanage port -l | grep http** и перещапускаем службу nginx **systemctl restart nginx**

![Image alt](/image/selinux8.png)

Удалить нестандартный порт из имеющегося типа можно с помощью команды: **semanage port -d -t http_port_t -p tcp 4881**

![Image alt](/image/selinux9.png)

### Способ 3
#### Разрешим в SELinux работу nginx на порту TCP 4881 c помощью формирования и установки модуля SELinux:
Запустим nginx **systemctl start nginx**
```
[root@selinux vagrant]# systemctl start nginx
Job for nginx.service failed because the control process exited with error code. See "systemctl status nginx.service" and "journalctl -xe" for details.
```
Nginx не запустился,так как SELinux продолжает его блокировать. Посмотрим логи SELinux, которые относятся к nginx: 
```
[root@selinux vagrant]# grep nginx /var/log/audit/audit.log
....
type=SYSCALL msg=audit(1676738964.624:1491): arch=c000003e syscall=49 success=no exit=-13 a0=6 a1=55bed4481858 a2=10 a3=7fffe3c82dd0 items=0 ppid=1 pid=15161 auid=4294967295 uid=0 gid=0 euid=0 suid=0 fsuid=0 egid=0 sgid=0 fsgid=0 tty=(none) ses=4294967295 comm="nginx" exe="/usr/sbin/nginx" subj=system_u:system_r:httpd_t:s0 key=(null)
type=SERVICE_START msg=audit(1676738964.639:1492): pid=1 uid=0 auid=4294967295 ses=4294967295 subj=system_u:system_r:init_t:s0 msg='unit=nginx comm="systemd" exe="/usr/lib/systemd/systemd" hostname=? addr=? terminal=? res=failed'
```
Воспользуемся утилитой **audit2allow** для того, чтобы на основе логов SELinux сделать модуль, разрешающий работу nginx на нестандартном порту: 
**grep nginx /var/log/audit/audit.log | audit2allow -M nginx**

![Image alt](/image/selinux10.png)

Audit2allow сформировал модуль, и сообщил нам команду, с помощью которой можно применить данный модуль: **semodule -i nginx.pp**
```
[root@selinux vagrant]# semodule -i nginx.pp
```
Попробуем снова запустить nginx: **systemctl start nginx**
```
[root@selinux vagrant]# systemctl start nginx
[root@selinux vagrant]# systemctl status nginx
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: active (running) since Сб 2023-02-18 17:34:18 UTC; 2h 50min ago
  Process: 15510 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
  Process: 15508 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
  Process: 15507 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
 Main PID: 15512 (nginx)
   CGroup: /system.slice/nginx.service
           ├─15512 nginx: master process /usr/sbin/nginx
           └─15513 nginx: worker process

фев 18 17:34:18 selinux systemd[1]: Starting The nginx HTTP and reverse proxy server...
фев 18 17:34:18 selinux nginx[15508]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
фев 18 17:34:18 selinux nginx[15508]: nginx: configuration file /etc/nginx/nginx.conf test is successful
фев 18 17:34:18 selinux systemd[1]: Started The nginx HTTP and reverse proxy server.
```
Чтобы просмотреть все установленные модули: **semodule -l**
Для удаления модуля воспользуемся командой: **semodule -r nginx**
```
[root@selinux vagrant]# semodule -r nginx
libsemanage.semanage_direct_remove_key: Removing last nginx module (no other nginx module exists at another priority).
```




# SELinux: проблема с удаленным обновлением зоны DNS

Подключемся к клиенту : **vagrant ssh client**
Вносим изменения в зону : **nsupdate -k /etc/named.zonetransfer.key**
```

