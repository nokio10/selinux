# selinux

## Задание
1. Запустить nginx на нестандартном порту 3-мя разными способами:

      переключатели setsebool;

      добавление нестандартного порта в имеющийся тип;

      формирование и установка модуля SELinux. 
      К сдаче:

      README с описанием каждого решения (скриншоты и демонстрация приветствуются).

2. Обеспечить работоспособность приложения при включенном selinux.

      развернуть приложенный стенд https://github.com/mbfx/otus-linux-adm/tree/master/selinux_dns_problems;
      выяснить причину неработоспособности механизма обновления зоны (см. README);
      предложить решение (или решения) для данной проблемы;
      выбрать одно из решений для реализации, предварительно обосновав выбор;
      реализовать выбранное решение и продемонстрировать его работоспособность. 
      К сдаче:
      README с анализом причины неработоспособности, возможными способами решения и обоснованием выбора одного из них;
      исправленный стенд или демонстрация работоспособной системы скриншотами и описанием.

## Задание №1 
## Запустить nginx на нестандартном порту 3-мя разными способами


Развернул машину командой "vagrant up", подключился к ней, убедился, что nginx не запустился еще раз )

### Способ №1 setsebool

```
Apr 27 06:54:58 selinux nginx[22263]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
Apr 27 06:54:58 selinux nginx[22263]: nginx: [emerg] bind() to [::]:4881 failed (13: Permission denied)
```

```
[vagrant@selinux ~]$ sudo -i
[root@selinux ~]#  systemctl status firewalld
● firewalld.service - firewalld - dynamic firewall daemon
   Loaded: loaded (/usr/lib/systemd/system/firewalld.service; disabled; vendor preset: enabled)
   Active: inactive (dead)
     Docs: man:firewalld(1)
[root@selinux ~]# getenforce
Enforcing
[root@selinux ~]# cat /var/log/audit/audit.log | grep 4881
type=AVC msg=audit(1650984205.312:848): avc:  denied  { name_bind } for  pid=2959 comm="nginx" src=4881 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:unreserved_port_t:s0 tclass=tcp_socket permissive=0
type=AVC msg=audit(1651042494.179:1093): avc:  denied  { name_bind } for  pid=22249 comm="nginx" src=4881 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:unreserved_port_t:s0 tclass=tcp_socket permissive=0
type=AVC msg=audit(1651042498.070:1099): avc:  denied  { name_bind } for  pid=22263 comm="nginx" src=4881 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:unreserved_port_t:s0 tclass=tcp_socket permissive=0
[root@selinux ~]# grep 1651042498.070:1099 /var/log/audit/audit.log | audit2why
type=AVC msg=audit(1651042498.070:1099): avc:  denied  { name_bind } for  pid=22263 comm="nginx" src=4881 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:unreserved_port_t:s0 tclass=tcp_socket permissive=0

        Was caused by:
        The boolean nis_enabled was set incorrectly.
        Description:
        Allow nis to enabled

        Allow access by executing:
        # setsebool -P nis_enabled 1
[root@selinux ~]# setsebool -P nis_enabled 1
[root@selinux ~]# systemctl restart nginx
[root@selinux ~]# systemctl status nginx
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: active (running) since Wed 2022-04-27 08:17:33 UTC; 5s ago
  Process: 22550 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
  Process: 22548 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
  Process: 22547 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
 Main PID: 22552 (nginx)
   CGroup: /system.slice/nginx.service
           ├─22552 nginx: master process /usr/sbin/nginx
           └─22554 nginx: worker process

Apr 27 08:17:33 selinux systemd[1]: Starting The nginx HTTP and reverse proxy server...
Apr 27 08:17:33 selinux nginx[22548]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
Apr 27 08:17:33 selinux nginx[22548]: nginx: configuration file /etc/nginx/nginx.conf test is successful
Apr 27 08:17:33 selinux systemd[1]: Started The nginx HTTP and reverse proxy server.
```

![image](https://user-images.githubusercontent.com/98832702/166105298-82ebe786-d214-4259-9c40-26a5d9bb47dc.png)

### Способ №2 добавление нестандартного порта в имеющийся тип

```
[root@selinux ~]# semanage port -l | grep http
http_cache_port_t              tcp      8080, 8118, 8123, 10001-10010
http_cache_port_t              udp      3130
http_port_t                    tcp      80, 81, 443, 488, 8008, 8009, 8443, 9000
pegasus_http_port_t            tcp      5988
pegasus_https_port_t           tcp      5989
[root@selinux ~]# semanage port -a -t http_port_t -p tcp 4881
[root@selinux ~]# semanage port -l | grep http_port_t
http_port_t                    tcp      4881, 80, 81, 443, 488, 8008, 8009, 8443, 9000
pegasus_http_port_t            tcp      5988
[root@selinux ~]# systemctl restart nginx.service
[root@selinux ~]# systemctl status nginx.service
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: active (running) since Tue 2022-05-03 17:06:35 UTC; 23s ago
  Process: 3259 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
  Process: 3256 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
  Process: 3255 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
 Main PID: 3261 (nginx)
   CGroup: /system.slice/nginx.service
           ├─3261 nginx: master process /usr/sbin/nginx
           └─3263 nginx: worker process

May 03 17:06:35 selinux systemd[1]: Starting The nginx HTTP and reverse proxy server...
May 03 17:06:35 selinux nginx[3256]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
May 03 17:06:35 selinux nginx[3256]: nginx: configuration file /etc/nginx/nginx.conf test is successful
May 03 17:06:35 selinux systemd[1]: Started The nginx HTTP and reverse proxy server.
```

### Способ №3 формирование и установка модуля SELinux

```
[root@selinux ~]# semanage port -d -t http_port_t -p tcp 4881
[root@selinux ~]# systemctl restart nginx
Job for nginx.service failed because the control process exited with error code. See "systemctl status nginx.service" and "journalctl -xe" for details.
[root@selinux ~]# systemctl start nginx
Job for nginx.service failed because the control process exited with error code. See "systemctl status nginx.service" and "journalctl -xe" for details.
[root@selinux ~]# grep nginx /var/log/audit/audit.log
type=SOFTWARE_UPDATE msg=audit(1651596474.068:846): pid=2811 uid=0 auid=1000 ses=2 subj=unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023 msg='sw="nginx-filesystem-1:1.20.1-9.el7.noarch" sw_type=rpm key_enforce=0 gpg_res=1 root_dir="/" comm="yum" exe="/usr/bin/python2.7" hostname=? addr=? terminal=? res=success'
type=SOFTWARE_UPDATE msg=audit(1651596474.352:847): pid=2811 uid=0 auid=1000 ses=2 subj=unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023 msg='sw="nginx-1:1.20.1-9.el7.x86_64" sw_type=rpm key_enforce=0 gpg_res=1 root_dir="/" comm="yum" exe="/usr/bin/python2.7" hostname=? addr=? terminal=? res=success'
type=AVC msg=audit(1651596485.620:848): avc:  denied  { name_bind } for  pid=3043 comm="nginx" src=4881 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:unreserved_port_t:s0 tclass=tcp_socket permissive=0
type=SYSCALL msg=audit(1651596485.620:848): arch=c000003e syscall=49 success=no exit=-13 a0=6 a1=55d4e4fe9788 a2=10 a3=7ffdc1498140 items=0 ppid=1 pid=3043 auid=4294967295 uid=0 gid=0 euid=0 suid=0 fsuid=0 egid=0 sgid=0 fsgid=0 tty=(none) ses=4294967295 comm="nginx" exe="/usr/sbin/nginx" subj=system_u:system_r:httpd_t:s0 key=(null)
type=SERVICE_START msg=audit(1651596485.620:849): pid=1 uid=0 auid=4294967295 ses=4294967295 subj=system_u:system_r:init_t:s0 msg='unit=nginx comm="systemd" exe="/usr/lib/systemd/systemd" hostname=? addr=? terminal=? res=failed'
type=SERVICE_START msg=audit(1651597595.073:912): pid=1 uid=0 auid=4294967295 ses=4294967295 subj=system_u:system_r:init_t:s0 msg='unit=nginx comm="systemd" exe="/usr/lib/systemd/systemd" hostname=? addr=? terminal=? res=success'
type=SERVICE_STOP msg=audit(1651597660.549:916): pid=1 uid=0 auid=4294967295 ses=4294967295 subj=system_u:system_r:init_t:s0 msg='unit=nginx comm="systemd" exe="/usr/lib/systemd/systemd" hostname=? addr=? terminal=? res=failed'
type=AVC msg=audit(1651597660.581:917): avc:  denied  { name_bind } for  pid=3279 comm="nginx" src=4881 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:unreserved_port_t:s0 tclass=tcp_socket permissive=0
type=SYSCALL msg=audit(1651597660.581:917): arch=c000003e syscall=49 success=no exit=-13 a0=6 a1=55de03649788 a2=10 a3=7fff233a94f0 items=0 ppid=1 pid=3279 auid=4294967295 uid=0 gid=0 euid=0 suid=0 fsuid=0 egid=0 sgid=0 fsgid=0 tty=(none) ses=4294967295 comm="nginx" exe="/usr/sbin/nginx" subj=system_u:system_r:httpd_t:s0 key=(null)
type=SERVICE_START msg=audit(1651597660.589:918): pid=1 uid=0 auid=4294967295 ses=4294967295 subj=system_u:system_r:init_t:s0 msg='unit=nginx comm="systemd" exe="/usr/lib/systemd/systemd" hostname=? addr=? terminal=? res=failed'
type=AVC msg=audit(1651597687.509:919): avc:  denied  { name_bind } for  pid=3291 comm="nginx" src=4881 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:unreserved_port_t:s0 tclass=tcp_socket permissive=0
type=SYSCALL msg=audit(1651597687.509:919): arch=c000003e syscall=49 success=no exit=-13 a0=6 a1=560d0202e788 a2=10 a3=7ffc76f31a10 items=0 ppid=1 pid=3291 auid=4294967295 uid=0 gid=0 euid=0 suid=0 fsuid=0 egid=0 sgid=0 fsgid=0 tty=(none) ses=4294967295 comm="nginx" exe="/usr/sbin/nginx" subj=system_u:system_r:httpd_t:s0 key=(null)
type=SERVICE_START msg=audit(1651597687.519:920): pid=1 uid=0 auid=4294967295 ses=4294967295 subj=system_u:system_r:init_t:s0 msg='unit=nginx comm="systemd" exe="/usr/lib/systemd/systemd" hostname=? addr=? terminal=? res=failed'
[root@selinux ~]# grep nginx /var/log/audit/audit.log | audit2allow -M nginx
******************** IMPORTANT ***********************
To make this policy package active, execute:

semodule -i nginx.pp

[root@selinux ~]# semodule -i nginx.pp
[root@selinux ~]# systemctl start nginx
[root@selinux ~]# systemctl status nginx
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: active (running) since Tue 2022-05-03 17:13:23 UTC; 5s ago
  Process: 3320 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
  Process: 3317 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
  Process: 3316 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
 Main PID: 3322 (nginx)
   CGroup: /system.slice/nginx.service
           ├─3322 nginx: master process /usr/sbin/nginx
           └─3324 nginx: worker process

May 03 17:13:23 selinux systemd[1]: Starting The nginx HTTP and reverse proxy server...
May 03 17:13:23 selinux nginx[3317]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
May 03 17:13:23 selinux nginx[3317]: nginx: configuration file /etc/nginx/nginx.conf test is successful
May 03 17:13:23 selinux systemd[1]: Started The nginx HTTP and reverse proxy server.
```

