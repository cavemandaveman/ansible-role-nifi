# Ansible Role: NiFi

An Ansible Role that installs [NiFi](https://nifi.apache.org/) on Linux.

## Requirements

None.

## Role Variables

Any key/value pair from a config file can be added to the following dicts:

```yaml
nifi_properties
bootstrap
logback
login_identity_providers
state_management
authorizers
zookeeper
```

The current config options for these files can be found [here](https://github.com/apache/nifi/blob/master/nifi-nar-bundles/nifi-framework-bundle/nifi-framework/nifi-resources/src/main/resources/conf).

## Dependencies

None.

## Example Playbook

```yaml
- hosts: all
  roles:
    - cavemandaveman.nifi
```

## License

GPLv3

## Author Information

This role was created in 2018 by cavemandaveman.
