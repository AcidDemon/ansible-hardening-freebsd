# ansible-hardening-freebsd

![Ansible](https://img.shields.io/badge/Ansible-2.15%2B-EE0000?logo=ansible&logoColor=white)
![FreeBSD](https://img.shields.io/badge/FreeBSD-14%20%2F%2015-AB2B28?logo=freebsd&logoColor=white)
![Hardening](https://img.shields.io/badge/security-hardening-4B275F)
![2FA](https://img.shields.io/badge/2FA-TOTP%20%2B%20FIDO2-005571)
![Tested with Vagrant](https://img.shields.io/badge/tested%20with-Vagrant-1868F2?logo=vagrant&logoColor=white)

[![License: MIT](https://img.shields.io/github/license/AcidDemon/ansible-hardening-freebsd?color=blue)](LICENSE)
[![Last commit](https://img.shields.io/github/last-commit/AcidDemon/ansible-hardening-freebsd)](https://github.com/AcidDemon/ansible-hardening-freebsd/commits/main)
[![Repo size](https://img.shields.io/github/repo-size/AcidDemon/ansible-hardening-freebsd)](https://github.com/AcidDemon/ansible-hardening-freebsd)
[![GitLab mirror](https://img.shields.io/badge/mirror-GitLab-FC6D26?logo=gitlab&logoColor=white)](https://git.inviziblenet.work/AcidDemon/ansible-hardening-freebsd)

Hardens a freshly installed FreeBSD host in one idempotent run: accounts, SSH and 2FA,
sshguard and CrowdSec on pf, OpenBSM audit, AIDE, Lynis, chkrootkit, and a daily posture
report. Runs after `ansible-base-freebsd`, which owns pf, users, and time.

## Roles

| Role | What it does |
|---|---|
| `accounts` | login.conf policy (umask, passwd_format, expiry), sudo hardening drop-in |
| `ssh_hardening` | `sshd_config.d` drop-in, AllowGroups self-heal, tight crypto, ECDSA excluded by algorithm |
| `crowdsec` | agent + pf firewall bouncer, whitelists `management_cidrs`, installs the sshd/linux collections |
| `sshguard` | log-driven SSH banning through base's pf `<sshguard>` table |
| `sysctl_hardening` | posture sysctls layered over base's baseline |
| `auditd` | OpenBSM audit classes |
| `aide` | file-integrity database with a daily periodic check |
| `lynis` | on-demand system audit |
| `hardening_misc` | coredumps off, cron/at allowlists, optional noexec `/tmp` and mail relay |
| `rootkit_scanners` | chkrootkit |
| `security_reporting` | daily posture report (sockstat, pfctl, cscli, pkg audit) |
| `ssh_2fa` | TOTP and FIDO2 tiers; the deploy account stays key-only |

## Run
    ansible-galaxy collection install -r requirements.yml
    ansible-playbook -i inventory/hosts.yml site.yml

Chain order is base, then app, then hardening. In a service repo, end `site.yml` with
`import_playbook: acidnetworks.hardening_freebsd.harden`. Set `management_cidrs` in
`inventory/group_vars/all/main.yml` to your control host's network, or the SSH cutover
locks you out.

## Lockout safety

`harden_safe_rollback` (default true) schedules an `at(1)` job before the SSH cutover. If
the run loses the session, that job strips the SSH drop-ins and reloads sshd after
`harden_rollback_minutes` (default 20). It gets cancelled once connectivity is re-checked.
A probe runs after ssh_hardening and after ssh_2fa, and every sshd drop-in is checked with
`sshd -t -f` before it lands. The rollback never touches pf.

## 2FA enrollment

2FA deploys with `ssh_2fa_totp_nullok: true`, so a user without an enrolled token is not
locked out. The deploy account is always key-only and exempt. Each operator enrolls before
you set nullok to false.

TOTP, on the host as the operator:

    google-authenticator --time-based --disallow-reuse --force --rate-limit=3 --rate-time=30 --window-size=3

FIDO2, on the workstation, then add the pubkey to the host and the user to `fido2users`:

    ssh-keygen -t ed25519-sk -f ~/.ssh/id_ed25519_sk

Once everyone is enrolled, set `ssh_2fa_totp_nullok: false` and re-run.

### Service accounts

`nullok` is what keeps unattended accounts working before that flip, and it stops
protecting them the moment you set it false. Anything that authenticates by key and
cannot answer a TOTP prompt - backup ingest, monitoring, CI - belongs in
`ssh_2fa_exempt_users` or `ssh_2fa_exempt_groups`, which render their own key-only
`Match` stanzas. `deploy_user` is always exempt. Such accounts also need to be in
`ssh_allow_group_members`, or `AllowGroups` shuts them out before 2FA is even reached.

## Jail hosts

A jail's sshd logs to the jail's own `auth.log`, which neither watcher sees by default.
Add the paths explicitly: `sshguard_log_paths` (appended to sshguard's `LOGREADER`) and
`crowdsec_acquis_paths` (written to `acquis.d/10-extra.yaml`). Bans still land in base's
shared pf tables, so a ban applies host-wide, jails included.

`ssh_client_alive_interval` / `ssh_client_alive_count_max` override the idle reaper.
The default kills a session after ~10 minutes without SSH-layer traffic, which is short
for a long `zfs recv` or a stalled backup.

## Lockout recovery

From the console:

    rm -f /etc/ssh/sshd_config.d/10-hardening.conf /etc/ssh/sshd_config.d/20-2fa.conf
    service sshd reload

pf belongs to base. If it is blocking you, re-run `ansible-base-freebsd`. `pfctl -d`
disables pf entirely as a last resort.
