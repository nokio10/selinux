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

### Способ №3 формирование и установка модуля SELinux
