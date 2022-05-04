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


Развернул машину командой "vagrant up" в директории /selinux, подключился к ней, убедился, что nginx не запустился еще раз )

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

## Задание №2
## Обеспечить работоспособность приложения при включенном selinux
Развернул машины командой "vagrant up" в директории selinux/2/selinux_dns_problems
### Способ №1 Изменение типа контекста безопасности для каталога /etc/named

```
root@ubuntu:~/selinux/otus-linux-adm/selinux_dns_problems# vagrant status
Current machine states:

ns01                      running (virtualbox)
client                    running (virtualbox)

This environment represents multiple VMs. The VMs are all listed
above with their current state. For more information about a specific
VM, run `vagrant status NAME`.
[vagrant@client ~]$ nsupdate -k /etc/named.zonetransfer.key
>
> ^C[vagrant@client ~]$ nsupdate -k /etc/named.zonetransfer.key
> server 192.168.50.10
> zone ddns.lab
> update add www.ddns.lab. 60 A 192.168.50.15
> send
update failed: SERVFAIL
> quit
[vagrant@client ~]$ sudo -i
[root@client ~]# cat /var/log/audit/audit.log | audit2why
root@ubuntu:~/selinux/otus-linux-adm/selinux_dns_problems# vagrant ssh ns01
Last login: Tue May  3 17:19:06 2022 from 10.0.2.2
[vagrant@ns01 ~]$ sudo -i
[root@ns01 ~]# cat /var/log/audit/audit.log | audit2why
type=AVC msg=audit(1651598750.373:1936): avc:  denied  { create } for  pid=5204 comm="isc-worker0000" name="named.ddns.lab.view1.jnl" scontext=system_u:system_r:named_t:s0 tcontext=system_u:object_r:etc_t:s0 tclass=file permissive=0

        Was caused by:
                Missing type enforcement (TE) allow rule.

                You can use audit2allow to generate a loadable module to allow this access.
[root@ns01 ~]# ls -laZ /etc/named
drw-rwx---. root named system_u:object_r:etc_t:s0       .
drwxr-xr-x. root root  system_u:object_r:etc_t:s0       ..
drw-rwx---. root named unconfined_u:object_r:etc_t:s0   dynamic
-rw-rw----. root named system_u:object_r:etc_t:s0       named.50.168.192.rev
-rw-rw----. root named system_u:object_r:etc_t:s0       named.dns.lab
-rw-rw----. root named system_u:object_r:etc_t:s0       named.dns.lab.view1
-rw-rw----. root named system_u:object_r:etc_t:s0       named.newdns.lab
```
В логах видно, что ошибка в контексте безопасности. Вместо типа
named_t используется тип etc_t. Изменю контекст безопасности
```
[root@ns01 ~]# sudo semanage fcontext -l | grep named
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
[root@ns01 ~]# sudo chcon -R -t named_zone_t /etc/named
[root@ns01 ~]# ls -laZ /etc/named
drw-rwx---. root named system_u:object_r:named_zone_t:s0 .
drwxr-xr-x. root root  system_u:object_r:etc_t:s0       ..
drw-rwx---. root named unconfined_u:object_r:named_zone_t:s0 dynamic
-rw-rw----. root named system_u:object_r:named_zone_t:s0 named.50.168.192.rev
-rw-rw----. root named system_u:object_r:named_zone_t:s0 named.dns.lab
-rw-rw----. root named system_u:object_r:named_zone_t:s0 named.dns.lab.view1
-rw-rw----. root named system_u:object_r:named_zone_t:s0 named.newdns.lab
[root@client ~]# nsupdate -k /etc/named.zonetransfer.key
> server 192.168.50.10
> zone ddns.lab
> update add www.ddns.lab. 60 A 192.168.50.15
> send
> quit
[root@client ~]# dig www.ddns.lab

; <<>> DiG 9.11.4-P2-RedHat-9.11.4-26.P2.el7_9.9 <<>> www.ddns.lab
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 17244
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
;; WHEN: Tue May 03 17:31:13 UTC 2022
;; MSG SIZE  rcvd: 96
[root@client ~]# reboot
[root@client ~]# dig @192.168.50.10 www.ddns.lab

; <<>> DiG 9.11.4-P2-RedHat-9.11.4-26.P2.el7_9.9 <<>> @192.168.50.10 www.ddns.lab
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 34986
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
;; WHEN: Tue May 03 17:32:51 UTC 2022
;; MSG SIZE  rcvd: 96
```


### Способ №2 создание модуля с помощью semodule

Отключаю selinux на серверной машине и создаю модуль
```
[vagrant@client ~]$ sudo -i
[root@client ~]# nsupdate -k /etc/named.zonetransfer.key
> server 192.168.50.10
>  zone ddns.lab
> update add www.ddns.lab. 60 A 192.168.50.15
> send
update failed: SERVFAIL
> quit
[root@client ~]# reboot


[root@ns01 ~]# setenforce 0
[root@ns01 ~]# cat /var/log/audit/audit.log | audit2why
type=AVC msg=audit(1651698452.879:1940): avc:  denied  { create } for  pid=5207 comm="isc-worker0000" name="named.ddns.lab.view1.jnl" scontext=system_u:system_r:named_t:s0 tcontext=system_u:object_r:etc_t:s0 tclass=file permissive=0

        Was caused by:
                Missing type enforcement (TE) allow rule.

                You can use audit2allow to generate a loadable module to allow this access.

type=AVC msg=audit(1651698505.123:1968): avc:  denied  { create } for  pid=5207 comm="isc-worker0000" name="named.ddns.lab.view1.jnl" scontext=system_u:system_r:named_t:s0 tcontext=system_u:object_r:etc_t:s0 tclass=file permissive=1

        Was caused by:
                Missing type enforcement (TE) allow rule.

                You can use audit2allow to generate a loadable module to allow this access.

type=AVC msg=audit(1651698505.123:1968): avc:  denied  { write } for  pid=5207 comm="isc-worker0000" path="/etc/named/dynamic/named.ddns.lab.view1.jnl" dev="sda1" ino=67403473 scontext=system_u:system_r:named_t:s0 tcontext=system_u:object_r:etc_t:s0 tclass=file permissive=1

        Was caused by:
                Missing type enforcement (TE) allow rule.

                You can use audit2allow to generate a loadable module to allow this access.

[root@ns01 ~]# audit2allow -M namedd --debug < /var/log/audit/audit.log
******************** IMPORTANT ***********************
To make this policy package active, execute:

semodule -i namedd.pp

[root@ns01 ~]# semodule -i namedd.pp
[root@ns01 ~]# setenforce 1
[root@ns01 ~]# reboot

[root@client ~]# nsupdate -k /etc/named.zonetransfer.key
> server 192.168.50.10
> zone ddns.lab
> update add www.ddns.lab. 60 A 192.168.50.15
> send
> quit
[root@client ~]# dig @192.168.50.10 www.ddns.lab

; <<>> DiG 9.11.4-P2-RedHat-9.11.4-26.P2.el7_9.9 <<>> @192.168.50.10 www.ddns.lab
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 1937
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

;; Query time: 2 msec
;; SERVER: 192.168.50.10#53(192.168.50.10)
;; WHEN: Wed May 04 21:13:09 UTC 2022
;; MSG SIZE  rcvd: 96
```

Первый способ решения является приоритетным, так как устраняет причину неисправности, а 2 способ скорее является обходным путем, который предлагает утилита audit2why и дает группе named_zone_t доступ к etc_t, что может вызвать проблемы с безопасностью системы. 
