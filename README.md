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
    selinux:    Active: failed (Result: exit-code) since Wed 2022-04-27 13:07:36 UTC; 13ms ago
    selinux:   Process: 3560 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=1/FAILURE)
    selinux:   Process: 3553 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
    selinux:
    selinux: Apr 27 13:07:36 selinux systemd[1]: Starting The nginx HTTP and reverse proxy server...
    selinux: Apr 27 13:07:36 selinux nginx[3560]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
    selinux: Apr 27 13:07:36 selinux nginx[3560]: nginx: [emerg] bind() to [::]:4881 failed (13: Permission denied)
    selinux: Apr 27 13:07:36 selinux nginx[3560]: nginx: configuration file /etc/nginx/nginx.conf test failed
    selinux: Apr 27 13:07:36 selinux systemd[1]: nginx.service: Control process exited, code=exited status=1
    selinux: Apr 27 13:07:36 selinux systemd[1]: nginx.service: Failed with result 'exit-code'.
    selinux: Apr 27 13:07:36 selinux systemd[1]: Failed to start The nginx HTTP and reverse proxy server.
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
   Active: failed (Result: exit-code) since Wed 2022-04-27 13:13:30 UTC; 3min 17s ago
  Process: 3780 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=1/FAILURE)
  Process: 3770 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)

Apr 27 13:13:30 selinux systemd[1]: Starting The nginx HTTP and reverse proxy server...
Apr 27 13:13:30 selinux nginx[3780]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
Apr 27 13:13:30 selinux nginx[3780]: nginx: [emerg] bind() to [::]:4881 failed (13: Permission denied)
Apr 27 13:13:30 selinux nginx[3780]: nginx: configuration file /etc/nginx/nginx.conf test failed
Apr 27 13:13:30 selinux systemd[1]: nginx.service: Control process exited, code=exited status=1
Apr 27 13:13:30 selinux systemd[1]: nginx.service: Failed with result 'exit-code'.
Apr 27 13:13:30 selinux systemd[1]: Failed to start The nginx HTTP and reverse proxy server.

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
type=AVC msg=audit(1651066146.857:554): avc:  denied  { name_bind } for  pid=4316 comm="nginx" src=4881 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:unreserved_port_t:s0 tclass=tcp_socket permissive=0
```
```
[root@selinux ~]# grep 4881 /var/log/audit/audit.log | audit2why
type=AVC msg=audit(1651066146.857:554): avc:  denied  { name_bind } for  pid=4316 comm="nginx" src=4881 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:unreserved_port_t:s0 tclass=tcp_socket permissive=0

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
   Active: active (running) since Wed 2022-04-27 14:12:32 UTC; 6s ago
  Process: 4691 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
  Process: 4688 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
  Process: 4686 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
 Main PID: 4692 (nginx)
    Tasks: 3 (limit: 4951)
   Memory: 14.0M
   CGroup: /system.slice/nginx.service
           ├─4692 nginx: master process /usr/sbin/nginx
           ├─4693 nginx: worker process
           └─4694 nginx: worker process

Apr 27 14:12:32 selinux systemd[1]: Starting The nginx HTTP and reverse proxy server...
Apr 27 14:12:32 selinux nginx[4688]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
Apr 27 14:12:32 selinux nginx[4688]: nginx: configuration file /etc/nginx/nginx.conf test is successful
Apr 27 14:12:32 selinux systemd[1]: Started The nginx HTTP and reverse proxy server.
```

```
1
```
