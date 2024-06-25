### OTUS Linux Professional Lesson #25 | Subject: Основы сбора и хранения логов

#### Цель домашнего задания
Научится проектировать централизованный сбор логов. Рассмотреть особенности разных платформ для сбора логов

#### Описание домашнего задания
1. В Vagrant разворачиваем 2 виртуальные машины web и log
2. На web настраиваем nginx  
3. На log настраиваем центральный лог сервер на любой системе на выбор
  - journald;
  - rsyslog;
  - elk.
4. Настраиваем аудит, следящий за изменением конфигов nginx 

Все критичные логи с web должны собираться и локально и удаленно.
Все логи с nginx должны уходить на удаленный сервер (локально только критичные).
Логи аудита должны также уходить на удаленную систему.

#### Инструкция по выполнению
1. Создаем виртуальные машины
```
$ vagrant up
```
2. Заходим на web-сервер
```
$ vagrant ssh web
$ sudo -i
```
3. Для правильной работы c логами, нужно, чтобы на всех хостах было настроено одинаковое время. Проверяем:
```console
root@web:~# timedatectl
               Local time: Tue 2024-06-25 17:26:32 UTC
           Universal time: Tue 2024-06-25 17:26:32 UTC
                 RTC time: Tue 2024-06-25 17:26:32
                Time zone: Etc/UTC (UTC, +0000)
System clock synchronized: no
              NTP service: inactive
          RTC in local TZ: no
```
Видим что NTP неактивно. Настриваем синхронизацию времени.
Синхронизацию времени можно настроить с помощью ntpd, chrony и systemd-timesyncd. Первый считается устаревшим, второй используется в более сложных схемах чем простая синхронизация времени с удаленным ntp сервером. Поэтому будет настраивать systemd-timesyncd. 

Вносим в /etc/systemd/timesyncd.conf список серверов, с которыми будем синхронизироваться:
```
[Time]
NTP=0.ru.pool.ntp.org, 1.ru.pool.ntp.org 2.ru.pool.ntp.org 3.ru.pool.ntp.org
...
...
```
Включаем синхронизацию:
```
root@web:~# timedatectl set-ntp true
```
После выполнения данной команды должна стартануть служба systemd-timesyncd:
```
root@web:~# systemctl status systemd-timesyncd
● systemd-timesyncd.service - Network Time Synchronization
     Loaded: loaded (/lib/systemd/system/systemd-timesyncd.service; enabled; vendor preset: enabled)
     Active: active (running) since Tue 2024-06-25 17:46:14 UTC; 36s ago
       Docs: man:systemd-timesyncd.service(8)
   Main PID: 3186 (systemd-timesyn)
     Status: "Initial synchronization to time server 91.206.16.3:123 (1.ru.pool.ntp.org)."
      Tasks: 2 (limit: 1586)
     Memory: 1.3M
        CPU: 31ms
     CGroup: /system.slice/systemd-timesyncd.service
             └─3186 /lib/systemd/systemd-timesyncd

Jun 25 17:46:14 web systemd[1]: Starting Network Time Synchronization...
Jun 25 17:46:14 web systemd[1]: Started Network Time Synchronization.
Jun 25 17:46:24 web systemd-timesyncd[3186]: Timed out waiting for reply from 91.207.136.55:123 (1.ru.pool.ntp.org).
Jun 25 17:46:25 web systemd-timesyncd[3186]: Initial synchronization to time server 91.206.16.3:123 (1.ru.pool.ntp.org).
```
Проверить статус синхронизации:
```
root@web:~# timedatectl timesync-status
       Server: 91.206.16.3 (1.ru.pool.ntp.org)
Poll interval: 1min 4s (min: 32s; max 34min 8s)
         Leap: normal
      Version: 4
      Stratum: 1
    Reference: GPS
    Precision: 1us (-24)
Root distance: 991us (max: 5s)
       Offset: -854us
        Delay: 90.875ms
       Jitter: 322us
 Packet count: 2
    Frequency: +0.000ppm

```
Меняем временную зону:
```
root@web:~# timedatectl set-timezone Europe/Moscow
```
Проверяем дату и время.

Тоже самое проделываем на втором сервере - log.

