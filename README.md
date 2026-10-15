# ansible-role-keepalived

An Ansible role that gives a group of hosts a shared virtual IP address with keepalived (VRRP), on RHEL, AlmaLinux and Rocky Linux 9 and 10. It is built to sit in front of Kibana, Logstash and Fleet Server, so users, senders and agents reach one name whichever instance is serving.

What it does:

- Runs one VRRP instance per virtual address. A host can carry several addresses, and several groups can share one run.
- Keeps an address only on a host whose service passes a health check (TCP connect, HTTP or HTTPS status, or your own command). A host that fails its check gives the address up.
- Works out each host's priority from its place in the inventory group, so the first host is preferred and nothing has to be set per host.
- Leaves the address where it is when a preferred host recovers (`nopreempt`), unless you ask for it to move back.
- Uses multicast by default, or unicast to the other hosts for networks that drop multicast.
- Checks the settings on every host, and compares them across hosts, before changing anything: clashing router ids, hosts that disagree on an address, equal priorities, missing interfaces.
- Tests the configuration with `keepalived --config-test` before it replaces the running one, and reloads rather than restarts.
- After a change, waits until each address answers on its service port.
- Accepts VRRP in firewalld, and trusts its health check helper in fapolicyd.

## Requirements

- **Control node:** ansible-core 2.15 or later, plus the `ansible.posix` collection (1.5.2 or later):

  ```bash
  ansible-galaxy collection install -r requirements.yml
  ```

- **Target hosts:** RHEL, AlmaLinux or Rocky Linux 9 or 10 (see [Supported platforms](#supported-platforms)). keepalived comes from AppStream, so the hosts need that repository or a mirror of it. On STIG-hardened hosts see [Hardened hosts](#hardened-hosts-fips-selinux-fapolicyd).
- **One layer 2 network** per address: the hosts sharing it must be on the same subnet, and the address must be free in it. Clouds that filter addresses a NIC does not own usually need their own mechanism instead of VRRP.

## Supported platforms

| Distribution | `ansible_facts['distribution']` | Major versions |
|---|---|---|
| Red Hat Enterprise Linux | `RedHat` | 9, 10 |
| AlmaLinux | `AlmaLinux` | 9, 10 |
| Rocky Linux | `Rocky` | 9, 10 |

Preflight stops the run on any other distribution or major version. To try another Enterprise Linux 9 or 10 rebuild, add its distribution fact to `keepalived_supported_distributions`. Those aren't tested.

## Installation

Add the role to your project's `requirements.yml`:

```yaml
roles:
  - name: keepalived
    src: https://github.com/jmaughmer/ansible-role-keepalived.git
    scm: git
    version: main          # or a release tag

collections:
  - name: ansible.posix
    version: ">=1.5.2"
```

```bash
ansible-galaxy install -r requirements.yml
```

## Quick start

[`examples/`](examples/) puts one address in front of each pair of Kibana, Logstash and Fleet Server hosts:

```
examples/
├── ansible.cfg                      # SSH settings for slow links, roles/collections paths
├── requirements.yml                 # this role + ansible.posix
├── playbook.yml
└── inventory/
    ├── hosts.yml                    # kibana, logstash and fleet_server pairs
    └── group_vars/
        ├── all/network.yml          # retries/timeouts for satellite links
        ├── kibana/main.yml          # kibana.example.com, https check on /api/status
        ├── logstash/main.yml        # logstash.example.com, tcp check on 5514
        └── fleet_server/main.yml    # fleet.example.com, https check on /api/status
```

```bash
cd examples
ansible-galaxy install -r requirements.yml
ansible-playbook playbook.yml
```

To run the examples against your working copy instead of the published role, link it into place instead of installing it:

```bash
mkdir -p roles && ln -s ../.. roles/keepalived
```

Useful tags: `keepalived_config`, `keepalived_service`, `keepalived_system`.

## Defining addresses

Each entry in `keepalived_instances` is one virtual address. Every host that shares it lists the same entry, so the natural place for it is the `group_vars` of the group that serves it:

```yaml
# group_vars/kibana/main.yml
keepalived_instances:
  - name: kibana
    address: 192.0.2.10/24
    virtual_router_id: 51
    check:
      type: https
      port: 5601
      path: /api/status
```

| Key | Required | Default | |
|---|---|---|---|
| `name` | yes | | Letters, digits, `_`, `.` and `-`; unique on the host. Names the VRRP instance. |
| `address` | yes | | IPv4 or IPv6 address, with an optional prefix length. Unique across instances. |
| `virtual_router_id` | yes | | 1-255. Unique per address family among all VRRP routers on the network, including ones this role does not manage. |
| `interface` | | `keepalived_interface` (the default route's interface) | Interface the address is added to. |
| `group` | | the group named like the instance, if there is one | Inventory group of the hosts sharing the address. Gives the order of preference and the unicast peers. Without one, the hosts in the play that list the instance share it. |
| `priority` | | from the group order | 1-254; the highest healthy host holds the address. |
| `preempt` | | `keepalived_preempt` (`false`) | |
| `advert_int` | | `keepalived_advert_int` (1 s) | |
| `check` | | `keepalived_check_defaults` | See [Health checks](#health-checks). |

**Hosts in more than one group.** `group_vars` of two groups would each set `keepalived_instances`, and only one wins. Give each group's address its own variable and collect them in the play, as the [site playbook](https://github.com/jmaughmer/ansible-elastic-stack) does:

```yaml
# group_vars/kibana/keepalived.yml
site_vrrp_kibana:
  name: kibana
  address: 192.0.2.10/24
  virtual_router_id: 51
  check: {type: tcp, port: 5601}

# playbooks/keepalived.yml
- name: Deploy keepalived
  hosts: kibana:logstash:fleet_server
  become: true
  any_errors_fatal: true
  vars:
    keepalived_instances: >-
      {{ [site_vrrp_kibana | default(none), site_vrrp_logstash | default(none),
          site_vrrp_fleet_server | default(none)] | select | list }}
  roles:
    - role: keepalived
```

## Health checks

keepalived runs the check every `interval` seconds as the unprivileged `keepalived_script_user`. After `fall` failures in a row the host leaves the election and gives the address up; after `rise` passes it takes part again.

| `type` | Passes when | Keys |
|---|---|---|
| `tcp` (default) | a TCP connection to `host:port` opens | `port`, `host` |
| `http` | `GET http://host:port/path` answers 2xx or 3xx | `port`, `host`, `path` |
| `https` | the same over TLS | `port`, `host`, `path`, `validate_certs`, `ca_path` |
| `script` | `command` exits 0 | `command` (an executable root owns, with arguments; no shell, no quotes) |
| `none` | always; the address follows VRRP alone | |

- `host` defaults to the host's own address on the instance's interface, which works whether the service listens on that address or on `0.0.0.0`.
- `https` doesn't verify the certificate by default: the check connects by address, which the certificate rarely names, and only asks whether the service answers. Set `validate_certs: true` (and `ca_path` for a private CA) with a `host` the certificate names.
- `interval`, `timeout`, `fall` and `rise` default to 2, 3, 2 and 2 (`keepalived_check_defaults`), so an address moves about 4 to 7 seconds after its service stops.
- **If every host fails its check, nobody holds the address.** That is the point of the check, but it means a service outage also takes the address away.

`tcp`, `http` and `https` run [`ansible-keepalived-check`](files/ansible-keepalived-check), a small bash helper the role installs in `/usr/libexec/keepalived`.

## Which host holds the address

Every instance starts as BACKUP on every host, and the healthy host with the highest priority wins the election. Unless `priority` or `keepalived_priority` is set, a host's priority is

```
keepalived_priority_base - keepalived_priority_step × (its position in the group)
```

that is 150, 140, 130... in inventory order. Preflight refuses hosts that would share a priority.

**Preemption.** With `keepalived_preempt: false` (the default, `nopreempt`) an address moves only when the host holding it fails. A preferred host that comes back does not take it back, which saves clients a second interruption. With `true` it moves back as soon as the preferred host is healthy again.

**Configuration changes** are applied with a reload, which keeps the address where it is unless the change itself moves it, for example a new priority with preemption on.

## Multicast or unicast

By default advertisements go to the VRRP multicast group (224.0.0.18, or ff02::12 for IPv6). Where multicast doesn't get through, set `keepalived_unicast: true`: each host then sends to the others directly.

- A host's address is its own address on the instance's interface, of the same family as the virtual address, or `keepalived_unicast_src_ip` when set.
- The peers are the other hosts in the instance's `group`. A peer outside the play (with `--limit`) has no facts, so give it `keepalived_unicast_src_ip` in its `host_vars`, or run without `--limit`. Preflight says which peer is missing.

## Running on part of the hosts

Priorities and peers come from the inventory group, so `--limit` runs keep them as long as every instance has a group, by name or by `group`. For instances without one, they would come from the hosts in the play and change, so preflight refuses a limited run until you set `group`, or `keepalived_allow_limit: true`.

## Using it with the Elastic roles

Run it after the services, so the addresses come up on hosts whose service already answers. In the [site playbook](https://github.com/jmaughmer/ansible-elastic-stack) it is the play after Fleet Server and before the agents.

- **Certificates.** Clients verify the name behind the address, so add it to each host's certificate. With [ansible-role-elastic-tls](https://github.com/jmaughmer/ansible-role-elastic-tls), list it in that group's `elastic_tls_certificates` SANs.
- **Kibana:** set `kibana_server_public_base_url` to the address's name. An `https` check on `/api/status` follows Kibana's own readiness: it answers 503 while starting or migrating.
- **Fleet Server:** put the address's name in `elastic_fleet_public_urls`, so agents enroll and connect through it. Its `/api/status` answers 200 only while it is healthy.
- **Logstash:** senders that take only one destination, such as syslog devices, send to the address. The Logstash role opens its own ports; list them in `keepalived_firewall_ports` as well if you run this role on its own.
- **Elasticsearch** doesn't need it: its clients take a list of nodes.

## Firewall

With `keepalived_manage_firewall` (on by default) the role accepts VRRP (IP protocol 112) in firewalld, permanently and immediately, in `keepalived_firewall_zone`:

| Variable | Default | |
|---|---|---|
| `keepalived_manage_firewall` | `true` | Set to `false` to manage the firewall yourself |
| `keepalived_firewall_zone` | `public` | Zone the rules go in |
| `keepalived_firewall_ports` | `[]` | Service ports reached through the addresses, e.g. `["5601/tcp"]` |

If firewalld is not running the role says what still has to be allowed rather than failing.

## Hardened hosts (FIPS, SELinux, fapolicyd)

The role is meant to run on hosts with the DISA STIG applied.

- **fapolicyd.** Ansible itself has to be allowed first; see the [Kibana role's README](https://github.com/jmaughmer/ansible-role-kibana#fapolicyd) for the rule. Beyond that, fapolicyd stops bash reading scripts rpm didn't install, so with `keepalived_fapolicyd_trust: auto` the role adds the health check helper to the trust database (`/etc/fapolicyd/trust.d/ansible-keepalived`) when fapolicyd is running, and rewrites the entry when the helper changes. A `script` check's own command has to be trusted too.
- **SELinux.** keepalived runs confined as `keepalived_t`. The helper lives in `/usr/libexec/keepalived`, which the policy labels `keepalived_unconfined_script_exec_t`, so checks can reach any port without changing booleans. A `script` check elsewhere runs confined, and may need `keepalived_connect_any`.
- **FIPS.** Nothing to do: VRRP version 3 has no authentication, and the checks only open connections.
- **Health checks run unprivileged** (`enable_script_security`, `script_user keepalived_script`), and keepalived refuses scripts in paths that user could change.

## Slow and unreliable networks

The role installs one package and otherwise changes a few files, so slow links mostly affect the download. It runs as an async job that survives a dropped SSH session, and is retried. For satellite links use the values in [`examples/inventory/group_vars/all/network.yml`](examples/inventory/group_vars/all/network.yml) and the SSH settings in [`examples/ansible.cfg`](examples/ansible.cfg).

## Notes and caveats

- **Check mode** is not supported for first-time installs, because the service doesn't exist yet. It works on hosts where keepalived is installed.
- **One address per instance.** Give a second address its own instance and router id.
- **VRRP version.** `keepalived_vrrp_version` defaults to 3, which carries IPv4 and IPv6. Use 2 only to match existing routers; IPv6 addresses need 3.
- **Removing an address:** drop its entry and run the role; the reload removes it from the host holding it.

## Variables

[`defaults/main.yml`](defaults/main.yml) documents every variable.

## Testing

[`molecule/default/`](molecule/default/) runs the role against three systemd containers on one container network: `ka-1` and `ka-2`, which share two addresses in front of a small web server on port 8080, and `ka-client`. The addresses are `.240` and `.241` of the container network, one with an `http` check and the inventory group `web`, the other with a `tcp` check and no group. The sequence is:

1. **converge**, then **idempotence**: a second converge must change nothing;
2. **check**: converge in check mode on the installed hosts, which must not fail;
3. **side effect** ([`side_effect.yml`](molecule/default/side_effect.yml)): the web server on the host holding the addresses is stopped, and both addresses must move to the other host; it is started again, and they must stay where they are;
4. **verify** ([`verify.yml`](molecule/default/verify.yml)): keepalived is running and enabled; the configuration passes `keepalived --config-test` and holds the expected priorities and checks; the helper belongs to root and runs as the script user; each address is held by exactly one host; and the client reaches the same web server through both.

Other scenarios:

| Scenario | What it covers |
|---|---|
| `unicast` | The default scenario with `keepalived_unicast: true`. |
| `preflight` | Settings preflight must refuse, each in its own check: an unsupported distribution, malformed instances, names, addresses and router ids, an unknown interface, IPv6 with VRRP version 2, a host outside its group, priorities out of range or equal, bad checks, clashing ids or addresses on a host and across hosts, hosts that disagree on an address, a unicast peer without an address, and bad firewall or fapolicyd settings. Nothing is installed. |
| `docker` | The default scenario on Docker instead of Podman. |

```bash
pip install -r molecule/requirements.txt
```

```bash
for s in default unicast preflight; do molecule test -s "$s" || break; done
```

The scenarios use Podman (rootless works), except `docker`, which needs Docker. keepalived needs `NET_ADMIN` and `NET_RAW` in the containers, which the scenarios grant. `MOLECULE_DISTRO` picks the image (default `rockylinux9`; also `rockylinux10`):

```bash
MOLECULE_DISTRO=rockylinux10 molecule test -s preflight
```

Lint with `yamllint .` and `ansible-lint` (configured by [`.yamllint`](.yamllint) and [`.ansible-lint`](.ansible-lint)). There is no GitHub Actions workflow; run the lint and Molecule scenarios locally.

Not covered by containers: firewalld rules, SELinux labels and fapolicyd trust. These need virtual machines with firewalld running, SELinux enforcing and fapolicyd running.

## Role layout

```
├── defaults/main.yml
├── vars/main.yml
├── meta/main.yml
├── files/ansible-keepalived-check   # tcp / http(s) health check helper
├── tasks/
│   ├── main.yml
│   ├── facts.yml                # resolves instances, group members, priorities, peers
│   ├── preflight.yml            # per host, then across hosts
│   ├── install.yml
│   ├── system.yml               # script user, helper, sysctl, firewalld
│   ├── fapolicyd.yml
│   ├── configure.yml            # sysconfig, keepalived.conf (config-tested)
│   ├── service.yml
│   └── wait_ready.yml           # each address answers on its port
├── templates/
├── examples/
└── molecule/                   # default, unicast, preflight (Podman) and docker scenarios
```

## Related roles

- [ansible-role-kibana](https://github.com/jmaughmer/ansible-role-kibana), [ansible-role-logstash](https://github.com/jmaughmer/ansible-role-logstash) and [ansible-role-elastic-fleet](https://github.com/jmaughmer/ansible-role-elastic-fleet): the services behind the addresses.
- [ansible-role-elastic-tls](https://github.com/jmaughmer/ansible-role-elastic-tls): issues certificates that name the addresses.
- [ansible-elastic-stack](https://github.com/jmaughmer/ansible-elastic-stack): the site playbook that runs them all.

## License

MIT
