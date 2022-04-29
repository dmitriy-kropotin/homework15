# homework15
1. Запускаю вирутальную машину

```
[tesla@rocky8 homework15]$ vagrant up
Bringing machine 'selinux' up with 'virtualbox' provider...
==> selinux: Importing base box 'generic/rocky8'...
....
    selinux: Job for nginx.service failed because the control process exited with error code.
    selinux: See "systemctl status nginx.service" and "journalctl -xe" for details.
    selinux: ● nginx.service - The nginx HTTP and reverse proxy server
    selinux:    Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
    selinux:    Active: failed (Result: exit-code) since Thu 2022-04-28 10:51:55 UTC; 10ms ago
    selinux:   Process: 4421 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=1/FAILURE)
    selinux:   Process: 4409 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
    selinux:
    selinux: Apr 28 10:51:55 selinux systemd[1]: Starting The nginx HTTP and reverse proxy server...
    selinux: Apr 28 10:51:55 selinux nginx[4421]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
    selinux: Apr 28 10:51:55 selinux nginx[4421]: nginx: [emerg] bind() to 0.0.0.0:4881 failed (13: Permission denied)
    selinux: Apr 28 10:51:55 selinux nginx[4421]: nginx: configuration file /etc/nginx/nginx.conf test failed
    selinux: Apr 28 10:51:55 selinux systemd[1]: nginx.service: Control process exited, code=exited status=1
    selinux: Apr 28 10:51:55 selinux systemd[1]: nginx.service: Failed with result 'exit-code'.
    selinux: Apr 28 10:51:55 selinux systemd[1]: Failed to start The nginx HTTP and reverse proxy server.
The SSH command responded with a non-zero exit status. Vagrant
assumes that this means the command failed. The output for this command
should be in the log above. Please read the output to determine what
went wrong.
```
2. Проверяю статус nginx после запуска машины

```
[vagrant@selinux ~]$ sudo -i
[root@selinux ~]# systemctl status nginx
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: failed (Result: exit-code) since Thu 2022-04-28 10:51:55 UTC; 16min ago
  Process: 4421 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=1/FAILURE)
  Process: 4409 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)

Apr 28 10:51:55 selinux systemd[1]: Starting The nginx HTTP and reverse proxy server...
Apr 28 10:51:55 selinux nginx[4421]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
Apr 28 10:51:55 selinux nginx[4421]: nginx: [emerg] bind() to 0.0.0.0:4881 failed (13: Permission denied)
Apr 28 10:51:55 selinux nginx[4421]: nginx: configuration file /etc/nginx/nginx.conf test failed
Apr 28 10:51:55 selinux systemd[1]: nginx.service: Control process exited, code=exited status=1
Apr 28 10:51:55 selinux systemd[1]: nginx.service: Failed with result 'exit-code'.
Apr 28 10:51:55 selinux systemd[1]: Failed to start The nginx HTTP and reverse proxy server.

```
3. nginx не удалось стартовать, нет прав на bind на 4881 порту

4. Проверяю состояние SELinux и корректность конфигурации nginx
```
[root@selinux ~]# getenforce
Enforcing
[root@selinux ~]# nginx -t
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```
5. Проверяю лог audit.log на сообщения о порте 4881

```
[root@selinux ~]# grep 4881 /var/log/audit/audit.log
type=AVC msg=audit(1651143115.799:559): avc:  denied  { name_bind } for  pid=4421 comm="nginx" src=4881 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:unreserved_port_t:s0 tclass=tcp_socket permissive=0

```
6. SELinux говорит, что не разрешает биндить nginx порт не из резервированных 

7. Посмотрю, что предложит программа audit2why

```
[root@selinux ~]# grep 4881 /var/log/audit/audit.log | audit2why
type=AVC msg=audit(1651143115.799:559): avc:  denied  { name_bind } for  pid=4421 comm="nginx" src=4881 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:unreserved_port_t:s0 tclass=tcp_socket permissive=0

        Was caused by:
        The boolean nis_enabled was set incorrectly.
        Description:
        Allow nis to enabled

        Allow access by executing:
        # setsebool -P nis_enabled 1
```
8. Ну и сделаю, что она предложила. И перезапущу nginx

```
[root@selinux ~]# setsebool -P nis_enabled 1
[root@selinux ~]# systemctl start nginx
[root@selinux ~]# systemctl status nginx
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: active (running) since Thu 2022-04-28 11:09:51 UTC; 5s ago
  Process: 4684 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
  Process: 4682 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
  Process: 4680 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
 Main PID: 4685 (nginx)
    Tasks: 3 (limit: 4951)
   Memory: 8.3M
   CGroup: /system.slice/nginx.service
           ├─4685 nginx: master process /usr/sbin/nginx
           ├─4686 nginx: worker process
           └─4687 nginx: worker process

Apr 28 11:09:51 selinux systemd[1]: Starting The nginx HTTP and reverse proxy server...
Apr 28 11:09:51 selinux nginx[4682]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
Apr 28 11:09:51 selinux nginx[4682]: nginx: configuration file /etc/nginx/nginx.conf test is successful
Apr 28 11:09:51 selinux systemd[1]: Started The nginx HTTP and reverse proxy server.
```
9. nginx запустился, стартовая страница доступна

![Снимок экрана 2022-04-27 в 22 18 02](https://user-images.githubusercontent.com/98701086/165603069-fc5f880f-e851-4a86-9dec-1ccdd2d10811.png)

10. Переключаю nis_enabled обратно в off 

```
[root@selinux ~]# getsebool -a | grep nis_enabled
nis_enabled --> on
[root@selinux ~]# setsebool -P nis_enabled off
[root@selinux ~]#
```
11. Перезапускаю nginx, биндить порт 4881 ему опять запрещено

```
[root@selinux ~]# systemctl restart nginx
Job for nginx.service failed because the control process exited with error code.
See "systemctl status nginx.service" and "journalctl -xe" for details.
[root@selinux ~]# systemctl status nginx
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: failed (Result: exit-code) since Thu 2022-04-28 11:28:39 UTC; 10s ago
  Process: 4684 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
  Process: 4729 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=1/FAILURE)
  Process: 4725 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
 Main PID: 4685 (code=exited, status=0/SUCCESS)

Apr 28 11:28:39 selinux systemd[1]: nginx.service: Succeeded.
Apr 28 11:28:39 selinux systemd[1]: Stopped The nginx HTTP and reverse proxy server.
Apr 28 11:28:39 selinux systemd[1]: Starting The nginx HTTP and reverse proxy server...
Apr 28 11:28:39 selinux nginx[4729]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
Apr 28 11:28:39 selinux nginx[4729]: nginx: [emerg] bind() to 0.0.0.0:4881 failed (13: Permission denied)
Apr 28 11:28:39 selinux nginx[4729]: nginx: configuration file /etc/nginx/nginx.conf test failed
Apr 28 11:28:39 selinux systemd[1]: nginx.service: Control process exited, code=exited status=1
Apr 28 11:28:39 selinux systemd[1]: nginx.service: Failed with result 'exit-code'.
Apr 28 11:28:39 selinux systemd[1]: Failed to start The nginx HTTP and reverse proxy server.
```
12. Посмотрю список портов в списке http_port_t

```
[root@selinux ~]# semanage port -l | grep http
http_cache_port_t              tcp      8080, 8118, 8123, 10001-10010
http_cache_port_t              udp      3130
http_port_t                    tcp      80, 81, 443, 488, 8008, 8009, 8443, 9000
pegasus_http_port_t            tcp      5988
pegasus_https_port_t           tcp      5989
```
13. Добавляю порт 4881 в этот список

```
[root@selinux ~]# semanage port -l | grep http
http_cache_port_t              tcp      8080, 8118, 8123, 10001-10010
http_cache_port_t              udp      3130
http_port_t                    tcp      4881, 80, 81, 443, 488, 8008, 8009, 8443, 9000
pegasus_http_port_t            tcp      5988
pegasus_https_port_t           tcp      5989
```
14. И перезапускаю nginx

```
[root@selinux ~]# systemctl restart nginx
[root@selinux ~]# systemctl status nginx
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: active (running) since Thu 2022-04-28 11:31:22 UTC; 10ms ago
  Process: 4752 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
  Process: 4750 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
  Process: 4748 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
 Main PID: 4753 (nginx)
    Tasks: 3 (limit: 4951)
   Memory: 5.0M
   CGroup: /system.slice/nginx.service
           ├─4753 nginx: master process /usr/sbin/nginx
           ├─4754 nginx: worker process
           └─4755 nginx: worker process

Apr 28 11:31:22 selinux systemd[1]: Starting The nginx HTTP and reverse proxy server...
Apr 28 11:31:22 selinux nginx[4750]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
Apr 28 11:31:22 selinux nginx[4750]: nginx: configuration file /etc/nginx/nginx.conf test is successful
Apr 28 11:31:22 selinux systemd[1]: Started The nginx HTTP and reverse proxy server.
```
15. nginx запустился

![Screenshot from 2022-04-28 14-35-32](https://user-images.githubusercontent.com/98701086/165743741-f817b366-e5f4-45dd-827f-b38d9f7942db.png)

16. Удаляю порт 4881 из списка http_port_t
```
[root@selinux ~]# semanage port -d -t http_port_t -p tcp 4881
```
17. Перезапускаю nginx, SELinux его опять не пускат на порт 4881

```
[root@selinux ~]# systemctl restart nginx
[root@selinux ~]# systemctl status nginx
Job for nginx.service failed because the control process exited with error code.
See "systemctl status nginx.service" and "journalctl -xe" for details.
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: failed (Result: exit-code) since Thu 2022-04-28 11:38:59 UTC; 12ms ago
  Process: 4752 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
  Process: 4799 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=1/FAILURE)
  Process: 4797 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
 Main PID: 4753 (code=exited, status=0/SUCCESS)

Apr 28 11:38:59 selinux systemd[1]: Starting The nginx HTTP and reverse proxy server...
Apr 28 11:38:59 selinux nginx[4799]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
Apr 28 11:38:59 selinux nginx[4799]: nginx: [emerg] bind() to 0.0.0.0:4881 failed (13: Permission denied)
Apr 28 11:38:59 selinux nginx[4799]: nginx: configuration file /etc/nginx/nginx.conf test failed
Apr 28 11:38:59 selinux systemd[1]: nginx.service: Control process exited, code=exited status=1
Apr 28 11:38:59 selinux systemd[1]: nginx.service: Failed with result 'exit-code'.
Apr 28 11:38:59 selinux systemd[1]: Failed to start The nginx HTTP and reverse proxy server.
```
18. Теперь попробую создать модуль для SELinux. Смотрю audit.log

```
[root@selinux ~]# grep nginx /var/log/audit/audit.log
...
type=AVC msg=audit(1651145922.194:636): avc:  denied  { name_bind } for  pid=4790 comm="nginx" src=4881 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:unreserved_port_t:s0 tclass=tcp_socket permissive=0
type=SYSCALL msg=audit(1651145922.194:636): arch=c000003e syscall=49 success=no exit=-13 a0=8 a1=55b80d175838 a2=10 a3=7fff7dcb4770 items=0 ppid=1 pid=4790 auid=4294967295 uid=0 gid=0 euid=0 suid=0 fsuid=0 egid=0 sgid=0 fsgid=0 tty=(none) ses=4294967295 comm="nginx" exe="/usr/sbin/nginx" subj=system_u:system_r:httpd_t:s0 key=(null)ARCH=x86_64 SYSCALL=bind AUID="unset" UID="root" GID="root" EUID="root" SUID="root" FSUID="root" EGID="root" SGID="root" FSGID="root"
type=SERVICE_START msg=audit(1651145922.210:637): pid=1 uid=0 auid=4294967295 ses=4294967295 subj=system_u:system_r:init_t:s0 msg='unit=nginx comm="systemd" exe="/usr/lib/systemd/systemd" hostname=? addr=? terminal=? res=failed'UID="root" AUID="unset"
type=AVC msg=audit(1651145939.025:638): avc:  denied  { name_bind } for  pid=4799 comm="nginx" src=4881 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:unreserved_port_t:s0 tclass=tcp_socket permissive=0
type=SYSCALL msg=audit(1651145939.025:638): arch=c000003e syscall=49 success=no exit=-13 a0=8 a1=55c75e2ce7b8 a2=10 a3=7ffc8e2b8230 items=0 ppid=1 pid=4799 auid=4294967295 uid=0 gid=0 euid=0 suid=0 fsuid=0 egid=0 sgid=0 fsgid=0 tty=(none) ses=4294967295 comm="nginx" exe="/usr/sbin/nginx" subj=system_u:system_r:httpd_t:s0 key=(null)ARCH=x86_64 SYSCALL=bind AUID="unset" UID="root" GID="root" EUID="root" SUID="root" FSUID="root" EGID="root" SGID="root" FSGID="root"
type=SERVICE_START msg=audit(1651145939.033:639): pid=1 uid=0 auid=4294967295 ses=4294967295 subj=system_u:system_r:init_t:s0 msg='unit=nginx comm="systemd" exe="/usr/lib/systemd/systemd" hostname=? addr=? terminal=? res=failed'UID="root" AUID="unset"
```
19. Генерирую модуль nginx-port-4881 через программу audit2allow

```
[root@selinux ~]# grep 1651145939.025:638 /var/log/audit/audit.log | audit2allow -M nginx-port-4881
******************** IMPORTANT ***********************
To make this policy package active, execute:

semodule -i nginx-port-4881.pp
```
20. Устанавливаю модуль и перезапускаю nginx

```
[root@selinux ~]# semodule -i nginx-port-4881.pp
[root@selinux ~]# systemctl restart nginx
[root@selinux ~]# systemctl status nginx
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: active (running) since Thu 2022-04-28 11:46:26 UTC; 7s ago
  Process: 4846 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
  Process: 4844 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
  Process: 4841 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
 Main PID: 4847 (nginx)
    Tasks: 3 (limit: 4951)
   Memory: 5.0M
   CGroup: /system.slice/nginx.service
           ├─4847 nginx: master process /usr/sbin/nginx
           ├─4848 nginx: worker process
           └─4849 nginx: worker process

Apr 28 11:46:26 selinux systemd[1]: Starting The nginx HTTP and reverse proxy server...
Apr 28 11:46:26 selinux nginx[4844]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
Apr 28 11:46:26 selinux nginx[4844]: nginx: configuration file /etc/nginx/nginx.conf test is successful
Apr 28 11:46:26 selinux systemd[1]: Started The nginx HTTP and reverse proxy server.

```
21. nginx запустился

![Screenshot from 2022-04-28 14-48-49](https://user-images.githubusercontent.com/98701086/165745677-d1792537-d188-4975-b996-30cedb937d99.png)

22. Модуль в установленных. 
```
[root@selinux ~]# semodule -l | grep nginx
nginx-port-4881
```
23. Удаляю модуль и перезапускаю nginx

```
[root@selinux ~]# semodule -r nginx-port-4881
libsemanage.semanage_direct_remove_key: Removing last nginx-port-4881 module (no other nginx-port-4881 module exists at another priority).
[root@selinux ~]# systemctl restart nginx
Job for nginx.service failed because the control process exited with error code.
See "systemctl status nginx.service" and "journalctl -xe" for details.
[root@selinux ~]# systemctl status nginx
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: failed (Result: exit-code) since Thu 2022-04-28 11:52:05 UTC; 3s ago
  Process: 4846 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
  Process: 4874 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=1/FAILURE)
  Process: 4872 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
 Main PID: 4847 (code=exited, status=0/SUCCESS)

Apr 28 11:52:05 selinux systemd[1]: nginx.service: Succeeded.
Apr 28 11:52:05 selinux systemd[1]: Stopped The nginx HTTP and reverse proxy server.
Apr 28 11:52:05 selinux systemd[1]: Starting The nginx HTTP and reverse proxy server...
Apr 28 11:52:05 selinux nginx[4874]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
Apr 28 11:52:05 selinux nginx[4874]: nginx: [emerg] bind() to 0.0.0.0:4881 failed (13: Permission denied)
Apr 28 11:52:05 selinux nginx[4874]: nginx: configuration file /etc/nginx/nginx.conf test failed
Apr 28 11:52:05 selinux systemd[1]: nginx.service: Control process exited, code=exited status=1
Apr 28 11:52:05 selinux systemd[1]: nginx.service: Failed with result 'exit-code'.
Apr 28 11:52:05 selinux systemd[1]: Failed to start The nginx HTTP and reverse proxy server.
```
24. Запуститься на порту 4881 не удалось. 
25. Я думаю, что предпочтительней в данном случае делать модуль для SELinux. Потому, что состояние резервированных портов для nginx и состояние sebool могут сбросить

26. Запускаю стенд для второй части домашней работы. 

```
[tesla@rocky8 selinux_dns_problems]$ vagrant status
Current machine states:

ns01                      running (virtualbox)
client                    running (virtualbox)

This environment represents multiple VMs. The VMs are all listed
above with their current state. For more information about a specific
VM, run `vagrant status NAME`.
```

```
###############################
### Enjoy! ####################
###############################
[vagrant@client ~]$ nsupdate -k /etc/named.zonetransfer.key
> server 192.168.50.10
> zone ddns.lab
> update add www.ddns.lab. 60 A 192.168.50.15
> send
update failed: SERVFAIL
> quit
[vagrant@client ~]$
```

```
[vagrant@client ~]$ sudo cat /var/log/audit/audit.log | audit2why
[vagrant@client ~]$
```


```
[root@ns01 ~]# cat /var/log/audit/audit.log | audit2why
type=AVC msg=audit(1651154538.261:1895): avc:  denied  { create } for  pid=5069 comm="isc-worker0000" name="named.ddns.lab.view1.jnl" scontext=system_u:system_r:named_t:s0 tcontext=system_u:object_r:etc_t:s0 tclass=file permissive=0

        Was caused by:
                Missing type enforcement (TE) allow rule.

                You can use audit2allow to generate a loadable module to allow this access.
```

```
[root@ns01 ~]# ls -laZ /etc/named
drw-rwx---. root named system_u:object_r:etc_t:s0       .
drwxr-xr-x. root root  system_u:object_r:etc_t:s0       ..
drw-rwx---. root named unconfined_u:object_r:etc_t:s0   dynamic
-rw-rw----. root named system_u:object_r:etc_t:s0       named.50.168.192.rev
-rw-rw----. root named system_u:object_r:etc_t:s0       named.dns.lab
-rw-rw----. root named system_u:object_r:etc_t:s0       named.dns.lab.view1
-rw-rw----. root named system_u:object_r:etc_t:s0       named.newdns.lab
```

```
[root@ns01 ~]# semanage fcontext -l | grep named
/etc/rndc.*                                        regular file       system_u:object_r:named_conf_t:s0
/var/named(/.*)?                                   all files          system_u:object_r:named_zone_t:s0
/etc/unbound(/.*)?                                 all files          system_u:object_r:named_conf_t:s0
/var/run/bind(/.*)?                                all files          system_u:object_r:named_var_run_t:s0
/var/log/named.*                                   regular file       system_u:object_r:named_log_t:s0
/var/run/named(/.*)?                               all files          system_u:object_r:named_var_run_t:s0
/var/named/data(/.*)?                              all files          system_u:object_r:named_cache_t:s0
/dev/xen/tapctrl.*                                 named pipe         system_u:object_r:xenctl_t:s0
/var/run/unbound(/.*)?                             all files          system_u:object_r:named_var_run_t:s0
/var/lib/softhsm(/.*)?                             all files          system_u:object_r:named_cache_t:s0
/var/lib/unbound(/.*)?                             all files          system_u:object_r:named_cache_t:s0
/var/named/slaves(/.*)?                            all files          system_u:object_r:named_cache_t:s0
/var/named/chroot(/.*)?                            all files          system_u:object_r:named_conf_t:s0
/etc/named\.rfc1912.zones                          regular file       system_u:object_r:named_conf_t:s0
/var/named/dynamic(/.*)?                           all files          system_u:object_r:named_cache_t:s0
/var/named/chroot/etc(/.*)?                        all files          system_u:object_r:etc_t:s0
/var/named/chroot/lib(/.*)?                        all files          system_u:object_r:lib_t:s0
/var/named/chroot/proc(/.*)?                       all files          <<None>>
/var/named/chroot/var/tmp(/.*)?                    all files          system_u:object_r:named_cache_t:s0
/var/named/chroot/usr/lib(/.*)?                    all files          system_u:object_r:lib_t:s0
/var/named/chroot/etc/pki(/.*)?                    all files          system_u:object_r:cert_t:s0
/var/named/chroot/run/named.*                      all files          system_u:object_r:named_var_run_t:s0
/var/named/chroot/var/named(/.*)?                  all files          system_u:object_r:named_zone_t:s0
/usr/lib/systemd/system/named.*                    regular file       system_u:object_r:named_unit_file_t:s0
/var/named/chroot/var/run/dbus(/.*)?               all files          system_u:object_r:system_dbusd_var_run_t:s0
/usr/lib/systemd/system/unbound.*                  regular file       system_u:object_r:named_unit_file_t:s0
/var/named/chroot/var/log/named.*                  regular file       system_u:object_r:named_log_t:s0
/var/named/chroot/var/run/named.*                  all files          system_u:object_r:named_var_run_t:s0
/var/named/chroot/var/named/data(/.*)?             all files          system_u:object_r:named_cache_t:s0
/usr/lib/systemd/system/named-sdb.*                regular file       system_u:object_r:named_unit_file_t:s0
/var/named/chroot/var/named/slaves(/.*)?           all files          system_u:object_r:named_cache_t:s0
/var/named/chroot/etc/named\.rfc1912.zones         regular file       system_u:object_r:named_conf_t:s0
/var/named/chroot/var/named/dynamic(/.*)?          all files          system_u:object_r:named_cache_t:s0
/var/run/ndc                                       socket             system_u:object_r:named_var_run_t:s0
/dev/gpmdata                                       named pipe         system_u:object_r:gpmctl_t:s0
/dev/initctl                                       named pipe         system_u:object_r:initctl_t:s0
/dev/xconsole                                      named pipe         system_u:object_r:xconsole_device_t:s0
/usr/sbin/named                                    regular file       system_u:object_r:named_exec_t:s0
/etc/named\.conf                                   regular file       system_u:object_r:named_conf_t:s0
/usr/sbin/lwresd                                   regular file       system_u:object_r:named_exec_t:s0
/var/run/initctl                                   named pipe         system_u:object_r:initctl_t:s0
/usr/sbin/unbound                                  regular file       system_u:object_r:named_exec_t:s0
/usr/sbin/named-sdb                                regular file       system_u:object_r:named_exec_t:s0
/var/named/named\.ca                               regular file       system_u:object_r:named_conf_t:s0
/etc/named\.root\.hints                            regular file       system_u:object_r:named_conf_t:s0
/var/named/chroot/dev                              directory          system_u:object_r:device_t:s0
/etc/rc\.d/init\.d/named                           regular file       system_u:object_r:named_initrc_exec_t:s0
/usr/sbin/named-pkcs11                             regular file       system_u:object_r:named_exec_t:s0
/etc/rc\.d/init\.d/unbound                         regular file       system_u:object_r:named_initrc_exec_t:s0
/usr/sbin/unbound-anchor                           regular file       system_u:object_r:named_exec_t:s0
/usr/sbin/named-checkconf                          regular file       system_u:object_r:named_checkconf_exec_t:s0
/usr/sbin/unbound-control                          regular file       system_u:object_r:named_exec_t:s0
/var/named/chroot_sdb/dev                          directory          system_u:object_r:device_t:s0
/var/named/chroot/var/log                          directory          system_u:object_r:var_log_t:s0
/var/named/chroot/dev/log                          socket             system_u:object_r:devlog_t:s0
/etc/rc\.d/init\.d/named-sdb                       regular file       system_u:object_r:named_initrc_exec_t:s0
/var/named/chroot/dev/null                         character device   system_u:object_r:null_device_t:s0
/var/named/chroot/dev/zero                         character device   system_u:object_r:zero_device_t:s0
/usr/sbin/unbound-checkconf                        regular file       system_u:object_r:named_exec_t:s0
/var/named/chroot/dev/random                       character device   system_u:object_r:random_device_t:s0
/var/run/systemd/initctl/fifo                      named pipe         system_u:object_r:initctl_t:s0
/var/named/chroot/etc/rndc\.key                    regular file       system_u:object_r:dnssec_t:s0
/usr/share/munin/plugins/named                     regular file       system_u:object_r:services_munin_plugin_exec_t:s0
/var/named/chroot_sdb/dev/null                     character device   system_u:object_r:null_device_t:s0
/var/named/chroot_sdb/dev/zero                     character device   system_u:object_r:zero_device_t:s0
/var/named/chroot/etc/localtime                    regular file       system_u:object_r:locale_t:s0
/var/named/chroot/etc/named\.conf                  regular file       system_u:object_r:named_conf_t:s0
/var/named/chroot_sdb/dev/random                   character device   system_u:object_r:random_device_t:s0
/etc/named\.caching-nameserver\.conf               regular file       system_u:object_r:named_conf_t:s0
/usr/lib/systemd/systemd-hostnamed                 regular file       system_u:object_r:systemd_hostnamed_exec_t:s0
/var/named/chroot/var/named/named\.ca              regular file       system_u:object_r:named_conf_t:s0
/var/named/chroot/etc/named\.root\.hints           regular file       system_u:object_r:named_conf_t:s0
/var/named/chroot/etc/named\.caching-nameserver\.conf regular file       system_u:object_r:named_conf_t:s0
/var/named/chroot/lib64 = /usr/lib
/var/named/chroot/usr/lib64 = /usr/lib
```


```
[root@ns01 ~]# chcon -R -t named_zone_t /etc/named
[root@ns01 ~]# ls -laZ /etc/named
drw-rwx---. root named system_u:object_r:named_zone_t:s0 .
drwxr-xr-x. root root  system_u:object_r:etc_t:s0       ..
drw-rwx---. root named unconfined_u:object_r:named_zone_t:s0 dynamic
-rw-rw----. root named system_u:object_r:named_zone_t:s0 named.50.168.192.rev
-rw-rw----. root named system_u:object_r:named_zone_t:s0 named.dns.lab
-rw-rw----. root named system_u:object_r:named_zone_t:s0 named.dns.lab.view1
-rw-rw----. root named system_u:object_r:named_zone_t:s0 named.newdns.lab
```

```
[vagrant@client ~]$ nsupdate -k /etc/named.zonetransfer.key
> server 192.168.50.10
> zone ddns.lab
> update add www.ddns.lab. 60 A 192.168.50.15
> send
> quit
```


```
[vagrant@client ~]$ dig www.ddns.lab

; <<>> DiG 9.11.4-P2-RedHat-9.11.4-26.P2.el7_9.9 <<>> www.ddns.lab
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 2971
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 1, ADDITIONAL: 2

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;www.ddns.lab.                  IN      A

;; ANSWER SECTION:
www.ddns.lab.           60      IN      A       192.168.50.15

;; AUTHORITY SECTION:
ddns.lab.               3600    IN      NS      ns01.dns.lab.

;; ADDITIONAL SECTION:
ns01.dns.lab.           3600    IN      A       192.168.50.10

;; Query time: 1 msec
;; SERVER: 192.168.50.10#53(192.168.50.10)
;; WHEN: Thu Apr 28 14:14:17 UTC 2022
;; MSG SIZE  rcvd: 96
```

```
[vagrant@client ~]$ nsupdate -k /etc/named.zonetransfer.key
> server 192.168.50.10
> zone ddns.lab
> update add ff.ddns.lab. 60 A 192.168.50.15
> send
> quit
[vagrant@client ~]$ dig ff.ddns.lab

; <<>> DiG 9.11.4-P2-RedHat-9.11.4-26.P2.el7_9.9 <<>> ff.ddns.lab
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 33613
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 1, ADDITIONAL: 2

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;ff.ddns.lab.                   IN      A

;; ANSWER SECTION:
ff.ddns.lab.            60      IN      A       192.168.50.15

;; AUTHORITY SECTION:
ddns.lab.               3600    IN      NS      ns01.dns.lab.

;; ADDITIONAL SECTION:
ns01.dns.lab.           3600    IN      A       192.168.50.10

;; Query time: 0 msec
;; SERVER: 192.168.50.10#53(192.168.50.10)
;; WHEN: Thu Apr 28 14:37:40 UTC 2022
;; MSG SIZE  rcvd: 95

```

```

[root@ns01 named]# semanage fcontext -a -t named_zone_t "/etc/named(/.*)?"
[root@ns01 named]# sudo semanage fcontext -l | grep named_zone_t
/var/named(/.*)?                                   all files          system_u:object_r:named_zone_t:s0
/var/named/chroot/var/named(/.*)?                  all files          system_u:object_r:named_zone_t:s0
/etc/named(/.*)?                                   all files          system_u:object_r:named_zone_t:s0
[root@ns01 named]# restorecon -v -R /etc/named
[root@ns01 named]# ls -laZ
drw-rwx---. root named system_u:object_r:named_zone_t:s0 .
drwxr-xr-x. root root  system_u:object_r:etc_t:s0       ..
drw-rwx---. root named unconfined_u:object_r:named_zone_t:s0 dynamic
-rw-rw----. root named system_u:object_r:named_zone_t:s0 named.50.168.192.rev
-rw-rw----. root named system_u:object_r:named_zone_t:s0 named.dns.lab
-rw-rw----. root named system_u:object_r:named_zone_t:s0 named.dns.lab.view1
-rw-rw----. root named system_u:object_r:named_zone_t:s0 named.newdns.lab
```


```
[vagrant@client ~]$ nsupdate -k /etc/named.zonetransfer.key
> server 192.168.50.10
> zone ddns.lab
> update add www.ddns.lab. 60 A 192.168.50.15
> send
> quit
[vagrant@client ~]$ dig www.ddns.lab

; <<>> DiG 9.11.4-P2-RedHat-9.11.4-26.P2.el7_9.9 <<>> www.ddns.lab
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 61100
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 1, ADDITIONAL: 2

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;www.ddns.lab.                  IN      A

;; ANSWER SECTION:
www.ddns.lab.           60      IN      A       192.168.50.15

;; AUTHORITY SECTION:
ddns.lab.               3600    IN      NS      ns01.dns.lab.

;; ADDITIONAL SECTION:
ns01.dns.lab.           3600    IN      A       192.168.50.10

;; Query time: 0 msec
;; SERVER: 192.168.50.10#53(192.168.50.10)
;; WHEN: Fri Apr 29 08:15:57 UTC 2022
;; MSG SIZE  rcvd: 96
```
