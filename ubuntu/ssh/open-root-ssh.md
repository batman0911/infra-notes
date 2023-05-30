# Open ssh for root user
- create file `my.conf` in `/etc/ssh/sshd_config.d/` with content
``` shell
PermitRootUser yes
```