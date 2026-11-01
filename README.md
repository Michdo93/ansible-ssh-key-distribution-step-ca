# Ansible SSH Key Distribution – step-ca variant

Let an account on one host log in to accounts on other hosts – with a **short-lived SSH certificate from [step-ca](https://smallstep.com/docs/step-ca/)** instead of a public key copied into every `authorized_keys`.

Typical use: the `openhab` service account on the openHAB VM may run commands as `pi` on several Raspberry Pis.

This is the certificate counterpart of [ansible-ssh-key-distribution](https://github.com/Michdo93/ansible-ssh-key-distribution). It builds on [ansible-inventory-generator-step-ca](https://github.com/Michdo93/ansible-inventory-generator-step-ca): the control node already talks to step-ca, and the managed hosts already trust the SSH user CA and carry host certificates.

| | Key distribution | This repository |
| --- | --- | --- |
| Source account gets | a key pair | a key pair **plus a certificate** (default 24 h), re-issued every 6 hours |
| Targets store | the public key in `authorized_keys` | one line – the principal – in `/etc/ssh/auth_principals/<account>` |
| Who talks to the CA | – | only the control node; source and targets need no step CLI and no CA credentials |
| Fingerprints on the source | target host keys copied into `known_hosts` | one `@cert-authority` line for the host CA |
| Revoking | remove the key everywhere | remove the principal (immediate) – any leftover certificate expires within 24 h |

## How it works

```text
                step-ca (CA host)
                  ▲   │ certificate for principal "openhab.openhab-vm"
     sign request │   ▼
   control node (Ansible) ──────────────► openhab-vm: ~openhab/.ssh/id_ed25519-cert.pub
          │                                           ~openhab/.ssh/known_hosts (@cert-authority)
          └─────────────────────────────► pi1, pi2:   /etc/ssh/auth_principals/pi += openhab.openhab-vm

   openhab@openhab-vm ── ssh pi@pi1 ──► accepted: certificate signed by the user CA, principal listed for pi
```

1. On the source host, the key pair of `source_user` is generated, or the existing one is reused.
2. The control node signs the public key with step-ca. The certificate's only principal is `cert_principal` (default `<source_user>.<source_host>`). The certificate is installed as `<key>-cert.pub`, which `ssh` picks up automatically.
3. Each target account lists that principal in its `AuthorizedPrincipalsFile`. The targets already trust the user CA, so nothing else changes on them.
4. The host CA goes into the source account's `known_hosts`, so there is no fingerprint prompt.
5. A login from every source to every target account is tested (`ssh -o BatchMode=yes`, run as the source account).
6. The distribution is recorded in `~/.config/ansible-step-ca/distributions.json`, and a cron job on the control node runs `renew.yml` every 6 hours.

If the CA host is managed by `ansible-inventory-generator-step-ca` (it was set up there with `ca-setup.yml`), the principal is added to the CA's SSH user policy automatically. It is also listed in `/etc/step-ca/extra-principals`, which `ca-setup.yml` keeps on every later policy sync. With an external CA, allow the principal in its policy yourself (`authority.policy.ssh.user.allow.principal` in `ca.json`).

## Requirements

- A control node set up with `ansible-inventory-generator-step-ca`. The playbooks read `~/.config/ansible-step-ca` (`config.env`, `provisioner-password`, `step/`) and use the same inventory and `ansible.cfg`.
- Target hosts enrolled by its `enroll.yml`. The playbook checks `sshd -T` for `TrustedUserCAKeys` and `AuthorizedPrincipalsFile /etc/ssh/auth_principals/%u` and stops with a hint otherwise.
- The source host must run Linux. Windows targets are supported but **not yet tested** (`ansible-galaxy collection install -r requirements.yml`).
- `cron` on the control node for automatic renewal.

## Usage

```bash
export ANSIBLE_CONFIG=/path/to/inventory/ansible.cfg

# openhab on openhab-vm may log in as pi on every other host
ansible-playbook distribute.yml -e source_host=openhab-vm -e source_user=openhab -e target_user=pi

# only the group "pis", accounts per host from key_targets, skip hosts without such an account
ansible-playbook distribute.yml -e source_host=openhab-vm -e source_user=openhab \
  -e target_hosts=pis -e skip_missing_users=true

# re-issue all registered certificates now (cron does this every 6 hours)
ansible-playbook renew.yml

# undo: targets stop accepting the principal, the certificate is deleted, renewal stops
ansible-playbook revoke.yml -e source_host=openhab-vm -e source_user=openhab -e target_user=pi
```

The target accounts follow the same rules as in `ansible-ssh-key-distribution`:

1. `-e target_user=...` for all targets,
2. otherwise the host's own `key_targets` (in `host_vars`/`group_vars`, a name or a list),
3. otherwise its `ansible_user`.

```yaml
# inventory/host_vars/pi1/keys.yml
key_targets: [pi, mqtt]
```

System accounts with `/usr/sbin/nologin` (like `openhab`) work as the source: outgoing `ssh` needs no login shell. The verification runs the client with `runuser -u <source_user> --`.

## Variables

| Variable | Default | Meaning |
| --- | --- | --- |
| `source_host` | – (required) | Inventory name of the source host |
| `source_user` | `ansible_user` of the source host | Account that owns the key pair and the certificate |
| `key_type` / `key_file_name` | `ed25519` / `id_<key_type>` | Key pair in `~source_user/.ssh/`; an existing key is reused |
| `cert_principal` | `<source_user>.<source_host>` | Principal in the certificate and on the targets. Must not contain `@`, because step-ca treats such values as e-mail addresses |
| `cert_duration` | `24h` | Certificate lifetime; must not exceed the provisioner's maximum (24 h with `ca-setup.yml`) |
| `target_hosts` | `all` | Receiving hosts; the source host is always excluded |
| `target_user` | – | Account on all targets (see the rules above) |
| `skip_missing_users` | `false` | Skip target accounts that do not exist instead of failing |
| `distribute_known_hosts` | `true` | Write the host CA into the source account's `known_hosts` |
| `verify_logins` | `true` | Test the logins after distributing |
| `install_renew_cron` | `true` | Install the renewal cron job on the control node |
| `renew_cron_minute` / `renew_cron_hour` | `23` / `*/6` | Schedule of `renew.yml` |

## Notes

- **Renewal**: certificates are re-issued every 6 hours and are valid for 24 hours, so a control node may be offline for up to about 18 hours before logins from the source account stop working. Running `renew.yml` (or `distribute.yml`) again restores them.
- **Revocation**: `revoke.yml` removes the principal from the targets, so logins are refused at once. It also deletes the certificate on the source host, takes the entry out of the registry and removes the principal from the managed CA policy. The cron job is removed when no distribution is left.
- **Several distributions**: each one has its own principal. The registry stores them all, and `renew.yml` renews every one of them.
- **Security**: the source account can log in to every listed target account, so prefer unprivileged target accounts. The certificate is only as safe as the source host. Access ends as soon as renewal stops or the principal is removed.

## Repository layout

```text
ansible-ssh-key-distribution-step-ca/
├── defaults.yml
├── distribute.yml        key pair + certificate, principals on targets, known_hosts, test, registry, cron
├── renew.yml             re-issue all registered certificates
├── revoke.yml            undo a distribution
├── requirements.yml      ansible.windows (Windows targets only)
└── tasks/
    ├── control_node.yml          read the step-ca configuration
    ├── ca_policy.yml             add/remove the principal in the CA policy (managed CA host)
    ├── source_cert.yml           key pair and certificate on the source host
    ├── target_linux.yml / target_linux_user.yml
    └── target_windows.yml / target_windows_revoke.yml
```

## Tested

Tested on Ubuntu 24.04 hosts (separate containers with their own sshd), using a CA set up by `ansible-inventory-generator-step-ca`:

- `openhab` (nologin system account) as the source, with targets `pi` on two hosts and a missing account skipped.
- Logins with strict host key checking and no prompt. The `openhab` certificate is refused for other accounts.
- Automatic CA policy update, and the policy sync by `ca-setup.yml` keeps the principal (`changed=0`).
- `renew.yml`, including the exact cron command line.
- `revoke.yml`: the login is refused immediately, and principal, certificate, registry entry, cron job and CA policy entry are removed.
- An external CA without the principal in its policy produces a clear error message.

Not tested: Windows targets.
