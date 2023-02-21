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
[vagrant@client ~]$ nsupdate -k /etc/named.zonetransfer.key 
> server 192.168.50.10
> zone ddns.lab 
> update add www.ddns.lab. 60 A 192.168.50.15
> send
update failed: SERVFAIL
> quit  
```
Как мы видим при обоновление зоны выдает ошибку: **update failed: SERVFAIL**. Посмотрим логи SELinux, чтобы понять в чем проблема. 
Воспользуемся утилитой **audit2why**
```
[vagrant@client ~]$ sudo su
[root@client vagrant]# cat /var/log/audit/audit.log | audit2why
```
На клиенте отсутсвуют ошибки. Подключимся к серверу **ns01** и проверим логи SELinux.
```
vagrant ssh ns01
Last login: Sat Feb 18 13:30:27 2023 from 10.0.2.2
[vagrant@ns01 ~]$ sudo -i
[root@ns01 ~]# cat /var/log/audit/audit.log | audit2why
type=AVC msg=audit(1677001292.566:839): avc:  denied  { create } for  pid=780 comm="isc-worker0000" name="named.ddns.lab.view1.jnl" scontext=system_u:system_r:named_t:s0 tcontext=system_u:object_r:etc_t:s0 tclass=file permissive=0

        Was caused by:
                Missing type enforcement (TE) allow rule.

                You can use audit2allow to generate a loadable module to allow this access.
```
В логах имеется ошибка в контексте безопасности. Вместо типа **named_t** стоит тип **etc_t**
Проверяем проблему в директории **/etc/named**
```
root@ns01 ~]# ls -laZ /etc/named
drw-rwx---. root named system_u:object_r:etc_t:s0       .
drwxr-xr-x. root root  system_u:object_r:etc_t:s0       ..
drw-rwx---. root named unconfined_u:object_r:etc_t:s0   dynamic
-rw-rw----. root named system_u:object_r:etc_t:s0       named.50.168.192.rev
-rw-rw----. root named system_u:object_r:etc_t:s0       named.dns.lab
-rw-rw----. root named system_u:object_r:etc_t:s0       named.dns.lab.view1
-rw-rw----. root named system_u:object_r:etc_t:s0       named.newdns.lab
```
Ошибка заключается в том, что контекст безопасности неправильный, конфигурационные файлы находятся в другой директории. Посмотрим в какой директории должны находится конфигурационные файлы с помощью команды: **sudo semanage fcontext -l | grep named**

```
sudo semanage fcontext -l | grep named
/etc/rndc.*                                        regular file       system_u:object_r:named_conf_t:s0 
/var/named(/.*)?                                   all files          system_u:object_r:named_zone_t:s0
```
Меняем контекст безопасности для директории **/etc/named**:**sudo chcon -R -t named_zone_t /etc/named**
```
[root@ns01 ~]# sudo chcon -R -t named_zone_t /etc/named
[root@ns01 ~]# ls -laZ /etc/named
drw-rwx---. root named system_u:object_r:named_zone_t:s0 .
drwxr-xr-x. root root  system_u:object_r:etc_t:s0       ..
drw-rwx---. root named unconfined_u:object_r:named_zone_t:s0 dynamic
-rw-rw----. root named system_u:object_r:named_zone_t:s0 named.50.168.192.rev
-rw-rw----. root named system_u:object_r:named_zone_t:s0 named.dns.lab
-rw-rw----. root named system_u:object_r:named_zone_t:s0 named.dns.lab.view1
-rw-rw----. root named system_u:object_r:named_zone_t:s0 named.newdns.lab
```
Переходим к клиенту и снова  вносим изменения зоны.
```
[root@client vagrant]# nsupdate -k /etc/named.zonetransfer.key
> server  192.168.50.10
> zone ddns.lab
> update add www.ddns.lab. 60 A 192.168.50.15 
> send
> quit
```
Проверяем наши изменения: **dig @192.168.50.10 www.ddns.lab**
```
[root@client vagrant]# dig @192.168.50.10 www.ddns.lab

; <<>> DiG 9.11.4-P2-RedHat-9.11.4-26.P2.el7_9.13 <<>> @192.168.50.10 www.ddns.lab
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 6071
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 1, ADDITIONAL: 2

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;www.ddns.lab.			IN	A

;; ANSWER SECTION:
www.ddns.lab.		60	IN	A	192.168.50.15

;; AUTHORITY SECTION:
ddns.lab.		3600	IN	NS	ns01.dns.lab.

;; ADDITIONAL SECTION:
ns01.dns.lab.		3600	IN	A	192.168.50.10

;; Query time: 0 msec
;; SERVER: 192.168.50.10#53(192.168.50.10)
;; WHEN: Вт фев 21 20:31:13 UTC 2023
;; MSG SIZE  rcvd: 96

```
### Итого:
Проблемой нашего стенда была в том, что SELinux блокировал доступ к обновлению фалов для DNS. В файлах была ошибка контекста безопасности. Файлы находились в другой директории. 
Данную проблему можно решить: 
+ Отключить SELinux 
+ Изменить тип контекста безопасности для директории: **/etc/named**
