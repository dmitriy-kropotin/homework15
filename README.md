# homework15

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

```
[root@selinux ~]# getenforce
Enforcing
[root@selinux ~]# nginx -t
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```

```
[root@selinux ~]# grep 4881 /var/log/audit/audit.log
type=AVC msg=audit(1651143115.799:559): avc:  denied  { name_bind } for  pid=4421 comm="nginx" src=4881 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:unreserved_port_t:s0 tclass=tcp_socket permissive=0

```
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

![Снимок экрана 2022-04-27 в 22 18 02](https://user-images.githubusercontent.com/98701086/165603069-fc5f880f-e851-4a86-9dec-1ccdd2d10811.png)

```
[root@selinux ~]# getsebool -a | grep nis_enabled
nis_enabled --> on
[root@selinux ~]# setsebool -P nis_enabled off
[root@selinux ~]#
```

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

```
[root@selinux ~]# semanage port -l | grep http
http_cache_port_t              tcp      8080, 8118, 8123, 10001-10010
http_cache_port_t              udp      3130
http_port_t                    tcp      80, 81, 443, 488, 8008, 8009, 8443, 9000
pegasus_http_port_t            tcp      5988
pegasus_https_port_t           tcp      5989
```


```
[root@selinux ~]# semanage port -l | grep http
http_cache_port_t              tcp      8080, 8118, 8123, 10001-10010
http_cache_port_t              udp      3130
http_port_t                    tcp      4881, 80, 81, 443, 488, 8008, 8009, 8443, 9000
pegasus_http_port_t            tcp      5988
pegasus_https_port_t           tcp      5989
```

```
[root@selinux ~]# systemctl restart nginx&&systemctl status nginx
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

