# homework15

```
[tesla@rocky8 homework15]$ vagrant up
Bringing machine 'selinux' up with 'virtualbox' provider...
==> selinux: Importing base box 'oraclelinux/8'...
....
    selinux: Complete!
    selinux: Job for nginx.service failed because the control process exited with error code.
    selinux: See "systemctl status nginx.service" and "journalctl -xe" for details.
    selinux: ● nginx.service - The nginx HTTP and reverse proxy server
    selinux:    Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
    selinux:    Active: failed (Result: exit-code) since Wed 2022-04-27 12:16:31 UTC; 11ms ago
    selinux:   Process: 5638 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=1/FAILURE)
    selinux:   Process: 5636 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
    selinux:
    selinux: Apr 27 12:16:31 selinux systemd[1]: Starting The nginx HTTP and reverse proxy server...
    selinux: Apr 27 12:16:31 selinux nginx[5638]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
    selinux: Apr 27 12:16:31 selinux nginx[5638]: nginx: [emerg] bind() to [::]:4881 failed (13: Permission denied)
    selinux: Apr 27 12:16:31 selinux nginx[5638]: nginx: configuration file /etc/nginx/nginx.conf test failed
    selinux: Apr 27 12:16:31 selinux systemd[1]: nginx.service: Control process exited, code=exited status=1
    selinux: Apr 27 12:16:31 selinux systemd[1]: nginx.service: Failed with result 'exit-code'.
    selinux: Apr 27 12:16:31 selinux systemd[1]: Failed to start The nginx HTTP and reverse proxy server.
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
   Active: failed (Result: exit-code) since Wed 2022-04-27 12:16:31 UTC; 31min ago
  Process: 5638 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=1/FAILURE)
  Process: 5636 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)

Apr 27 12:16:31 selinux systemd[1]: Starting The nginx HTTP and reverse proxy server...
Apr 27 12:16:31 selinux nginx[5638]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
Apr 27 12:16:31 selinux nginx[5638]: nginx: [emerg] bind() to [::]:4881 failed (13: Permission denied)
Apr 27 12:16:31 selinux nginx[5638]: nginx: configuration file /etc/nginx/nginx.conf test failed
Apr 27 12:16:31 selinux systemd[1]: nginx.service: Control process exited, code=exited status=1
Apr 27 12:16:31 selinux systemd[1]: nginx.service: Failed with result 'exit-code'.
Apr 27 12:16:31 selinux systemd[1]: Failed to start The nginx HTTP and reverse proxy server.

```

```
```
