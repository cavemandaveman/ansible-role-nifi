# Ansible Role: NiFi

An Ansible Role that installs [NiFi](https://nifi.apache.org/) on Linux. By default, it installs NiFi in a way that makes [upgrading](https://cwiki.apache.org/confluence/display/NIFI/1.x.0+to+1.x.0+Upgrade) painless.

## Requirements

Requires at least Java 8.

## Role Variables

See `defaults/main.yml` for all variables and how to specify them. For a deeper dive, the [NiFi System Administrator’s Guide](https://nifi.apache.org/docs/nifi-docs/html/administration-guide.html) is a great resource.

The following specifies where to install NiFi, along with a home directory (which will be symbolically linked to the install directory). Also, a centralized config directory to store files that need not be changed (to avoid copying during upgrades).

```yaml
nifi_config_dirs:
  install: /opt/nifi/releases
  home: /opt/nifi/releases/current
  external_config: /opt/nifi/config_resources
```

By default, this is the directory structure that will be created:

```text
|--opt/
  |--nifi/
    |--releases/
      |--current -> nifi-1.9.0/
      |--nifi-1.8.0/
      |--nifi-1.9.0/
    |--config_resources/
      |--archive/
      |--authorizations.xml
      |--content_repository/
      |--custom_nars/
      |--database_repository/
      |--flow.xml.gz
      |--flowfile_repository/
      |--provenance_repository/
      |--state/
      |--users.xml
```

Any key/value pair from a config file can be added to the following dicts. Dict names correspond to file names. The current config options for these files can be found [here](https://github.com/apache/nifi/blob/master/nifi-nar-bundles/nifi-framework-bundle/nifi-framework/nifi-resources/src/main/resources/conf).

```yaml
nifi_properties:
bootstrap:
logback:
login_identity_providers:
state_management:
authorizers:
zookeeper:
```

## Dependencies

None.

## Example Playbooks

These assume you have `hash_behaviour=merge` [set in your config](https://docs.ansible.com/ansible/latest/reference_appendices/config.html#default-hash-behaviour).

Basic single node NiFi instance:

```yaml
- hosts: nifi_servers
  become: yes
  roles:
    - cavemandaveman.nifi
```

Basic 3 node NiFi cluster using embedded Zookeeper:

```yaml
- hosts: nifi_servers
  become: yes
  roles:
    - cavemandaveman.nifi
  vars:
    nifi_properties:
      nifi.web.http.host: "{{ ansible_fqdn }}"
      nifi.web.http.port: 8080
      nifi.cluster.is.node: true
      nifi.cluster.node.address: "{{ ansible_fqdn }}"
      nifi.cluster.node.protocol.port: 11443
      nifi.cluster.flow.election.max.candidates: 3
      nifi.cluster.load.balance.host: "{{ ansible_fqdn }}"
      nifi.cluster.load.balance.port: 6342
      nifi.state.management.embedded.zookeeper.start: true
      nifi.zookeeper.connect.string: nifi_server1:2181,nifi_server2:2181,nifi_server3:2181
    state_management:
       /stateManagement/cluster-provider/property[@name="Connect String"]: "{{ nifi_properties['nifi.zookeeper.connect.string] }}"
    # Assuming nifi_server1 = 192.168.1.10, nifi_server2 = 192.168.1.11, nifi_server3 = 192.168.1.12
    # we have Ansible automatically set the myid file on each host to last octet of the node's IP address
    # and we set the last part of the zookeeper['server.x'] keys to those same numbers.
    zookeeper_myid: "{{ ansible_default_ipv4.address.split('.')[-1] }}"
    zookeeper:
      server.10: nifi_server1:2888:3888
      server.11: nifi_server2:2888:3888
      server.12: nifi_server3:2888:3888
```

## License

GPLv3

## Author Information

This role was created in 2018 by cavemandaveman.
