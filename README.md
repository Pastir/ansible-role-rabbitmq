# Ansible Role: pastir.rabbitmq

Installs RabbitMQ and Erlang from the Team RabbitMQ apt repositories
([apt quick start](https://www.rabbitmq.com/docs/install-debian#apt-quick-start)), pins and holds exact
versions, manages `rabbitmq.conf` / `rabbitmq-env.conf` / `advanced.config`, the Erlang cookie, cluster
membership, plugins, vhosts, users and policies.

Nothing restarts implicitly. Upgrades and restarts run one node at a time:
1. check that the node and every cluster member are running without alarms;
2. drain the node (`rabbitmq-upgrade drain`);
3. upgrade or restart it;
4. wait for the cluster, then `revive` the node.

If a node does not come back, its previous config is restored and the remaining nodes are not touched.

## Requirements

- Ubuntu 24.04 (noble) or 26.04 (resolute), amd64
- ansible-core >= 2.16, collection `community.rabbitmq` (`meta/requirements.yml`)
- Cluster nodes resolve each other's short hostnames (`rabbit@<hostname>`)

## Versions

| Variable | Default | |
|---|---|---|
| `rabbitmq_version` | `3.12.14` | `"3.12.14"` or full `"3.12.14-1"` (Ubuntu builds too: `"3.12.1-1ubuntu1.7"`). Empty = latest |
| `rabbitmq_erlang_version` | `26.2.5.13` | `"26.2.5.13"` or full `"1:26.2.5.13-1"` (`"1:25.3.2.8+dfsg-1ubuntu4.8"`). [Compatibility](https://www.rabbitmq.com/docs/which-erlang) |
| `rabbitmq_hold` | `true` | `dpkg --set-selections hold` |
| `rabbitmq_allow_upgrade` | `false` | Without it the role fails if installed versions differ (this includes replacing Ubuntu archive builds). The check runs before any change: a refused host keeps its sources, pins and packages |
| `rabbitmq_erlang_repo_suite` | `""` | Empty = host release. If the Erlang series is no longer built for that release, the nearest one is used (`vars/main.yml`) |

The Team RabbitMQ repository has only Erlang 27 for 26.04. Erlang 26 is therefore taken from the noble suite,
which was checked to work on 26.04. `/etc/apt/preferences.d/rabbitmq` pins `erlang*` and `rabbitmq-server` at
priority 990. This makes the pinned version the candidate even when newer builds exist, but apt never downgrades
on its own. The signing key (`0A9AF2115F4687BD29803A206B73A36E6026DFCA`) ships with the role. Set
`rabbitmq_repo_key_url` to download it instead.

## Configuration

| Variable | Default | |
|---|---|---|
| `rabbitmq_conf` | `{}` | `rabbitmq.conf`, `key: value`. `null` = not managed |
| `rabbitmq_env` | `{}` | `rabbitmq-env.conf`, `KEY: value`. `null` = not managed |
| `rabbitmq_advanced_config` | `""` | Raw Erlang terms. Empty = file removed, `null` = not managed |
| `rabbitmq_restart_on_change` | `false` | `false`: report only. A mark in `/var/lib/rabbitmq/.ansible-restart-pending` keeps the report on later runs. `true`: rolling restart |
| `rabbitmq_rolling_drain` | `true` | Use maintenance mode in a cluster |
| `rabbitmq_health_timeout` | `300` | Seconds to wait for the node and all members |
| `rabbitmq_plugins` | management, prometheus | List. The legacy `rabbitmq_plugins_enabled: "a,b"` string still works |
| `rabbitmq_plugins_exclusive` | `true` | Disable explicitly enabled plugins that are not listed |

A fresh node (installed by this run) is restarted right away.

## Cluster

| Variable | Default | |
|---|---|---|
| `rabbitmq_cluster_group` | `""` | Inventory group of one cluster. Empty = standalone |
| `rabbitmq_cluster_seed` | first host of the group | The others join it. Cluster-wide objects are applied there |
| `rabbitmq_erlang_cookie` | `""` | Required for a cluster (letters/digits, ≥ 20). Keep it in a vault |
| `rabbitmq_erlang_cookie_force` | `false` | Allow replacing the cookie of an installed node |
| `rabbitmq_cluster_name` | `""` | `set_cluster_name` |
| `rabbitmq_cluster_join_force` | `false` | A joining node is reset first. The role refuses this while the node holds messages |
| `rabbitmq_nodename` | `rabbit@<hostname>` | |

## Vhosts, users, policies

Applied on the seed node with `community.rabbitmq`. Passwords are checked on every run (`update_password: always`).

```yaml
rabbitmq_vhosts:
  - name: app
rabbitmq_users:
  - user: app
    password: "{{ vault_rabbitmq_app_password }}"
    tags: [management]
    permissions:
      - {vhost: app, configure_priv: ".*", read_priv: ".*", write_priv: ".*"}
  - user: guest
    state: absent
rabbitmq_policies:
  - {name: limit, vhost: app, pattern: "^limited\\.", apply_to: queues, tags: {max-length: 1000}}
rabbitmq_enable_all_feature_flags: false   # irreversible; needed before the next minor upgrade
```

## Example

```yaml
- hosts: rabbitmq
  become: true
  roles:
    - role: pastir.rabbitmq
      vars:
        rabbitmq_cluster_group: rabbitmq
        rabbitmq_erlang_cookie: "{{ vault_rabbitmq_cookie }}"
        rabbitmq_conf:
          listeners.tcp.default: 5672
          management.tcp.ip: 127.0.0.1
```

Upgrade, node by node:

```bash
ansible-playbook site.yml --limit rabbitmq -e rabbitmq_allow_upgrade=true
```

Run the upgrade on the whole cluster group, not with a `--limit` to single nodes. The health checks need to see
all members.

## Testing

`molecule test` covers:
- a cluster of 24.04 + 24.04 + 26.04 nodes;
- an extra 24.04 node with RabbitMQ 3.12.1 and Erlang 25 from the Ubuntu archive: refused without
  `rabbitmq_allow_upgrade` (apt sources, key and pins stay untouched), then upgraded by the role;
- a quorum queue message that survives a rolling restart after a config change;
- users, policies, feature flags, holds.

## Installation

```yaml
# requirements.yml
roles:
  - name: pastir.rabbitmq
    src: git+git@github.com:Pastir/ansible-role-rabbitmq.git
    version: main  # or a tag
```

```bash
ansible-galaxy install -r requirements.yml
```

## License

BSD-3-Clause
