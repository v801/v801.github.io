+++
author = "v801"
title = "Basic OpenSSH Hardening"
date = "2022-07-20"
tags = [
    "ssh",
    "network",
    "linux hardening",
]
+++
Basic setup for OpenSSH hardening on a new box.
<!--more-->

## Intro

The OpenSSH daemon, known as [sshd](https://linux.die.net/man/8/sshd), has config options we can set in `/etc/ssh/sshd_config` to help us secure it.  Changes to this file can be tested first in test mode with `sudo sshd -T`.

### Disable root SSH login
```
PermitRootLogin no
```

### Limit max number of auth attempts for session
```
MaxAuthTries 3
```

### Disable empty password logins
```
PermitEmptyPasswords no
```

### Disable other auth methods
```
ChallengeResponseAuthentication no
KerberosAuthentication no
GSSAPIAuthentication no
```

### Disable x11 forwarding
```
X11Forwarding no
```

### Disable passing custom environment vars
```
PermitUserEnvironment no
```

### Disable other tunneling and fowarding options
```
AllowAgentForwarding no
AllowTcpForwarding no
PermitTunnel no
```

### Allow users based on user/ip
```
AllowUsers [user or *]@ip
```

Once the options are set, run the test mode with `sudo sshd -T`.  If everything looks good, reload sshd with `sudo systemctl restart ssh`.

Thanks for reading!  In a future article we'll dive into more OpenSSH hardening options.
