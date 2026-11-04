# Changelog

## 2.0.0 — 2026-11

### BREAKING
- Defaults pin RabbitMQ 3.12.14 and Erlang 26.2.5.13 (were: latest). An installed node with other versions
  makes the role fail until `rabbitmq_allow_upgrade: true`.
- Internal `__rabbitmq_*` defaults are gone; use the documented `rabbitmq_*` variables.
  `rabbitmq_version` / `rabbitmq_erlang_version` / `rabbitmq_plugins_enabled` keep working.
- Sources: deb822 `rabbitmq-erlang.sources` / `rabbitmq-server.sources`. The key is in `/etc/apt/keyrings`.
  `sources.list.d/rabbitmq.list` is removed.
- The service is no longer restarted on every package task.
- Plugins not listed are disabled (`rabbitmq_plugins_exclusive`).

### Added
- Pin (priority 990) and hold. Erlang 26 on Ubuntu 26.04 comes from the noble suite.
- Rolling upgrade and restart: health gate, drain/revive, config rollback, stop after the first failed node.
- `rabbitmq.conf`, `rabbitmq-env.conf`, `advanced.config`. A pending restart mark is reported until the
  restart happens.
- Erlang cookie, cluster join (refuses to reset a node with messages), cluster name.
- Vhosts, users, policies, feature flags.
- `/usr/local/sbin/rabbitmq-wait-healthy`, molecule tests.
