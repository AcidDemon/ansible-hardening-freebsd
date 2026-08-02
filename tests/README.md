# Testing hardening_freebsd

The smoke runs the real chain: **base_freebsd → hardening_freebsd**.

## Prereq (host)

Install the base layer (local) + deps so `tests/chain.yml` resolves:

```sh
ansible-galaxy collection install -r tests/requirements.yml
```

(base_freebsd must be tagged `v0.1.0` in `../ansible-base-freebsd`.)

## Spin up + provision

```sh
vagrant up            # boots, applies base_freebsd THEN hardening_freebsd
vagrant provision     # re-run (idempotence check)
vagrant ssh           # inspect
vagrant destroy -f
```

`deploy_user` is pinned to `vagrant` so ssh_2fa leaves the vagrant connection key-only.
`management_cidrs` is pinned to the vagrant-libvirt subnet so the SSH lockout guards pass.

## Smoke assertions (inside `vagrant ssh`)

```sh
# SSH hardening
grep -q 'AllowGroups' /etc/ssh/sshd_config.d/10-hardening.conf && sshd -t && echo "sshd config ok"
id vagrant | grep -q sshusers && echo "connecting user self-healed into sshusers"
# sshguard + crowdsec on pf (tables declared by base)
service sshguard status; pfctl -t sshguard -T show 2>/dev/null || echo "(sshguard table empty is fine)"
sysrc -n sshguard_enable crowdsec_enable crowdsec_firewall_enable auditd_enable
# OpenBSM
service auditd status && auditctl -l 2>/dev/null; praudit -l /dev/auditpipe & sleep 1; kill %1 2>/dev/null
# integrity + reporting
ls -l /var/db/aide/aide.db 2>/dev/null; which lynis chkrootkit
ls /usr/local/etc/periodic/*/*.aide /usr/local/etc/periodic/daily/*security-report* 2>/dev/null
# account policy
grep -q umask /etc/login.conf && echo "login.conf policy present"
# rollback disarmed (no leftover at job)
atq
```

## Idempotence

```sh
vagrant provision 2>&1 | tail -20    # PLAY RECAP: changed=0 ideal (probe/command tasks excepted)
```
