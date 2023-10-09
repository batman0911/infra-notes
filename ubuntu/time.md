# Time util

Check time

```shell
timedatectl
```

List time zone

```shell
timedatectl list-timezones
```

Set +7 time zone

```shell
sudo timedatectl set-timezone Asia/Ho_Chi_Minh
```

Check mysql/mariadb time zone

```shell
SHOW GLOBAL VARIABLES LIKE 'time_zone';
```