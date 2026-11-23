Asuswrt Merlin DNS - Ansible Role
===

![AI Usage](https://img.shields.io/badge/AI_Usage-Tests_Only-blue)

Configure explicit domain name entries and expose them to selected VLANs on Asus routers running Asuswrt-Merlin firmware.

**Note:** this role was tested on RT-BE88U, RT-AX88U Pro, and RT-AX86U units only.


## Requirements
- Asus router(s) running Asuswrt-Merlin ≥ 3004.388.12_2.
- SSH access and JFFS partition enabled on router.


## Role Variables

| Variable               | Default | Description                                                                    |
| ---------------------- | ------- | ------------------------------------------------------------------------------ |
| `dns_inventory_groups` | `[]`    | Inventory hosts in each specified group will have a domain name entry created. |
| `dns_network_domain`   |         | Append the specified network domain to inventory host names (e.g. `prd-haos-01` from the inventory becomes `prd-haos-01.internal`).                   |
| `dns_add_bridges`      | `[]`    | Specify bridge devices (e.g. `br52`) to make domain name entries available to. Use the `brctl` on the router to find the bridge device name. |
| `dns_entries`          | `[]`    | Custom domain name entries. See below for an example.                          |


## Example Configuration

```ini
# inventory/hosts.ini

[routers]
rt_be88u  ansible_host=192.168.50.1
rt_ax86u  ansible_host=192.168.50.2

[web_servers]
prd-web-01  ansible_host=192.168.52.2
prd-web-02  ansible_host=192.168.52.3

[app_servers]
prd-haos-01  ansible_host=192.168.53.2

[infra]
prd-infra-01  ansible_host=192.168.50.3
```

```yaml
# inventory/group_vars/routers.yml

dns_inventory_groups:
  - web_servers
  - app_servers

dns_network_domain: internal

dns_add_bridges:
  - br52
  - br53

dns_entries:
  - domain_names:
      - ca.internal
      - nbde.internal
    target: "{{ hostvars['prd-infra-01']['ansible_host'] }}"
  - domain_names: [secure.example.com]
    target: "192.168.50.4"
```

```yaml
# playbooks/asuswrt-merlin-dns.yml

- name: asuswrt merlin dns config
  hosts: routers
  gather_facts: false
  become: false
  roles:
    - role: asuswrt_merlin_dns
```

## Roadmap

- Specify VLAN tags instead of bridge device names. This will be a breaking change in the next version.


## AI Usage Policy & Disclosure

- **Core Codebase:** Written manually without AI code generation. This will change as more features are added.
- **Test Suite:** AI tools were used to assist in writing tests in the `molecule/` directory.
- **Research & Debugging:** AI was used as a Q&A and troubleshooting reference during development; all architectural decisions and implementations are human-authored.
