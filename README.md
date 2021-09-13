# Ansible Role: NiFi

An Ansible Role that installs [NiFi](https://nifi.apache.org/) on Linux. By default, it installs NiFi in a way that makes [upgrading](https://cwiki.apache.org/confluence/display/NIFI/1.x.0+to+1.x.0+Upgrade) painless.

## Requirements

Requires at least Java 8.

## Role Variables

See `defaults/main.yml` for all variables and how to specify them. For a deeper dive, the [NiFi System Administrator’s Guide](https://nifi.apache.org/docs/nifi-docs/html/administration-guide.html) is a great resource.

The following specifies where to download (or look for existing) binaries (tarballs), where to install NiFi, and a home directory which will be symbolically linked to the specified release. Also, a centralized config directory to store files that need not be changed (to avoid copying during upgrades). You can add more artbitrary key/value pairs to this dict and those directories will be created. This might be useful if you need extra directories for things like custom nars, drivers, etc.

```yaml
nifi_config_dirs:
  binaries: /tmp
  install: /opt/nifi/releases
  home: /opt/nifi/releases/current
  external_config: /opt/nifi/config_resources
```

By default, this is the directory structure that will be created:

```text
|--opt/
  |--nifi/
    |--releases/
      |--current -> nifi-1.14.0/
      |--nifi-1.14.0/
      |--nifi-1.13.2/
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
nifi_env:
logback:
login_identity_providers:
state_management:
authorizers:
zookeeper:
```

## Dependencies

None.

## Example Playbooks

These assume you have `hash_behaviour=merge` [set in your config](https://docs.ansible.com/ansible/latest/reference_appendices/config.html#default-hash-behaviour). If not, please also include the default dict key/values from `defaults/main.yml`.

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
      # HTTP properties must be unset for HTTPS to work
      nifi.web.http.host: ""
      nifi.web.http.port: ""
      nifi.web.https.host: "{{ ansible_fqdn }}"
      nifi.web.https.port: 8443
      nifi.cluster.is.node: true
      nifi.cluster.node.address: "{{ ansible_fqdn }}"
      nifi.cluster.node.protocol.port: 11443
      nifi.cluster.flow.election.max.candidates: 3
      nifi.cluster.load.balance.host: "{{ ansible_fqdn }}"
      nifi.cluster.load.balance.port: 6342
      nifi.state.management.embedded.zookeeper.start: true
      nifi.zookeeper.connect.string: nifi_server1:2181,nifi_server2:2181,nifi_server3:2181
    login_identity_providers:
      /loginIdentityProviders/provider/identifier: single-user-provider
      /loginIdentityProviders/provider/class: org.apache.nifi.authentication.single.user.SingleUserLoginIdentityProvider
    authorizers_user_group_providers: 0
    authorizers:
      /authorizers/authorizer/identifier: single-user-authorizer
      /authorizers/authorizer/class: org.apache.nifi.authorization.single.user.SingleUserAuthorizer
    state_management:
      /stateManagement/cluster-provider/property[@name="Connect String"]: "{{ nifi_properties['nifi.zookeeper.connect.string'] }}"
    # Assuming nifi_server1 = 192.168.1.10, nifi_server2 = 192.168.1.11, nifi_server3 = 192.168.1.12
    # we have Ansible automatically set the myid file on each host to last octet of the node's IP address
    # and we set the 'X' of the zookeeper['server.X'] keys to those same numbers.
    zookeeper_myid: "{{ ansible_default_ipv4.address.split('.')[-1] }}"
    zookeeper:
      server.10: nifi_server1:2888:3888
      server.11: nifi_server2:2888:3888
      server.12: nifi_server3:2888:3888
```

Secure single node NiFi instance with LDAP:

```yaml
- hosts: nifi_servers
  become: yes
  roles:
    - cavemandaveman.nifi
  vars:
    nifi_properties:
      # HTTP properties must be unset for HTTPS to work
      nifi.web.http.host: ""
      nifi.web.http.port: ""
      nifi.web.https.host: "{{ ansible_fqdn }}"
      nifi.web.https.port: 8443
      nifi.security.keystore: /path/to/keystore.jks
      nifi.security.keystoreType: JKS
      nifi.security.keystorePasswd: keystorePassword
      nifi.security.keyPasswd: keyPassword
      nifi.security.truststore: /path/to/truststore.jks
      nifi.security.truststoreType: JKS
      nifi.security.truststorePasswd: truststorePassword
    login_identity_providers:
      /loginIdentityProviders/provider/identifier: ldap-provider
      /loginIdentityProviders/provider/class: org.apache.nifi.ldap.LdapProvider
      /loginIdentityProviders/provider/property[@name="Authentication Strategy"]: SIMPLE
      /loginIdentityProviders/provider/property[@name="Manager DN"]: cn=nifi,ou=people,dc=example,dc=com
      /loginIdentityProviders/provider/property[@name="Manager Password"]: password
      /loginIdentityProviders/provider/property[@name="Referral Strategy"]: FOLLOW
      /loginIdentityProviders/provider/property[@name="Connect Timeout"]: 10 secs
      /loginIdentityProviders/provider/property[@name="Read Timeout"]: 10 secs
      /loginIdentityProviders/provider/property[@name="Url"]: ldap://hostname:port
      /loginIdentityProviders/provider/property[@name="User Search Base"]: OU=people,DC=example,DC=com
      /loginIdentityProviders/provider/property[@name="User Search Filter"]: sAMAccountName={0}
      /loginIdentityProviders/provider/property[@name="Identity Strategy"]: USE_DN
      /loginIdentityProviders/provider/property[@name="Authentication Expiration"]: 12 hours
    authorizers_user_group_providers: 1
    authorizers:
      /authorizers/userGroupProvider[1]/identifier: file-user-group-provider
      /authorizers/userGroupProvider[1]/class: org.apache.nifi.authorization.FileUserGroupProvider
      /authorizers/userGroupProvider[1]/property[@name="Users File"]: "{{ nifi_config_dirs.external_config }}/users.xml"
      /authorizers/userGroupProvider[1]/property[@name="Initial User Identity 1"]: cn=John Smith,ou=people,dc=example,dc=com
      /authorizers/accessPolicyProvider/identifier: file-access-policy-provider
      /authorizers/accessPolicyProvider/class: org.apache.nifi.authorization.FileAccessPolicyProvider
      /authorizers/accessPolicyProvider/property[@name="User Group Provider"]: file-user-group-provider
      /authorizers/accessPolicyProvider/property[@name="Authorizations File"]: "{{ nifi_config_dirs.external_config }}/authorizations.xml"
      /authorizers/accessPolicyProvider/property[@name="Initial Admin Identity"]: cn=John Smith,ou=people,dc=example,dc=com
      /authorizers/authorizer/identifier: managed-authorizer
      /authorizers/authorizer/class: org.apache.nifi.authorization.StandardManagedAuthorizer
      /authorizers/authorizer/property[@name="Access Policy Provider"]: file-access-policy-provider
```

Secure 3 node NiFi cluster with LDAP using embedded zookeeper:

```yaml
- hosts: nifi_servers
  become: yes
  roles:
    - cavemandaveman.nifi
  vars:
    nifi_properties:
      # HTTP properties must be unset for HTTPS to work
      nifi.web.http.host: ""
      nifi.web.http.port: ""
      nifi.web.https.host: "{{ ansible_fqdn }}"
      nifi.web.https.port: 8443
      nifi.security.keystore: /path/to/keystore.jks
      nifi.security.keystoreType: JKS
      nifi.security.keystorePasswd: keystorePassword
      nifi.security.keyPasswd: keyPassword
      nifi.security.truststore: /path/to/truststore.jks
      nifi.security.truststoreType: JKS
      nifi.security.truststorePasswd: truststorePassword
      nifi.cluster.protocol.is.secure: true
      nifi.cluster.is.node: true
      nifi.cluster.node.address: "{{ ansible_fqdn }}"
      nifi.cluster.node.protocol.port: 11443
      nifi.cluster.flow.election.max.candidates: 3
      nifi.cluster.load.balance.host: "{{ ansible_fqdn }}"
      nifi.cluster.load.balance.port: 6342
      nifi.state.management.embedded.zookeeper.start: true
      nifi.zookeeper.connect.string: nifi_server1:2181,nifi_server2:2181,nifi_server3:2181
    login_identity_providers:
      /loginIdentityProviders/provider/identifier: ldap-provider
      /loginIdentityProviders/provider/class: org.apache.nifi.ldap.LdapProvider
      /loginIdentityProviders/provider/property[@name="Authentication Strategy"]: SIMPLE
      /loginIdentityProviders/provider/property[@name="Manager DN"]: cn=nifi,ou=people,dc=example,dc=com
      /loginIdentityProviders/provider/property[@name="Manager Password"]: password
      /loginIdentityProviders/provider/property[@name="Referral Strategy"]: FOLLOW
      /loginIdentityProviders/provider/property[@name="Connect Timeout"]: 10 secs
      /loginIdentityProviders/provider/property[@name="Read Timeout"]: 10 secs
      /loginIdentityProviders/provider/property[@name="Url"]: ldap://hostname:port
      /loginIdentityProviders/provider/property[@name="User Search Base"]: OU=people,DC=example,DC=com
      /loginIdentityProviders/provider/property[@name="User Search Filter"]: sAMAccountName={0}
      /loginIdentityProviders/provider/property[@name="Identity Strategy"]: USE_DN
      /loginIdentityProviders/provider/property[@name="Authentication Expiration"]: 12 hours
    authorizers_user_group_providers: 1
    authorizers:
      /authorizers/userGroupProvider[1]/identifier: file-user-group-provider
      /authorizers/userGroupProvider[1]/class: org.apache.nifi.authorization.FileUserGroupProvider
      /authorizers/userGroupProvider[1]/property[@name="Users File"]: "{{ nifi_config_dirs.external_config }}/users.xml"
      /authorizers/userGroupProvider[1]/property[@name="Initial User Identity 1"]: cn=John Smith,ou=people,dc=example,dc=com
      # Use the full DN of the node certificates here
      /authorizers/userGroupProvider[1]/property[@name="Initial User Identity 2"]: CN=nifi_server1.example.com, O=ExampleLLC, L=Saint Louis, ST=Missouri, C=US
      /authorizers/userGroupProvider[1]/property[@name="Initial User Identity 3"]: CN=nifi_server2.example.com, O=ExampleLLC, L=Saint Louis, ST=Missouri, C=US
      /authorizers/userGroupProvider[1]/property[@name="Initial User Identity 4"]: CN=nifi_server3.example.com, O=ExampleLLC, L=Saint Louis, ST=Missouri, C=US
      /authorizers/accessPolicyProvider/identifier: file-access-policy-provider
      /authorizers/accessPolicyProvider/class: org.apache.nifi.authorization.FileAccessPolicyProvider
      /authorizers/accessPolicyProvider/property[@name="User Group Provider"]: file-user-group-provider
      /authorizers/accessPolicyProvider/property[@name="Authorizations File"]: "{{ nifi_config_dirs.external_config }}/authorizations.xml"
      /authorizers/accessPolicyProvider/property[@name="Initial Admin Identity"]: cn=John Smith,ou=people,dc=example,dc=com
      /authorizers/accessPolicyProvider/property[@name="Node Identity 1"]: CN=nifi_server1.example.com, O=ExampleLLC, L=Saint Louis, ST=Missouri, C=US
      /authorizers/accessPolicyProvider/property[@name="Node Identity 2"]: CN=nifi_server2.example.com, O=ExampleLLC, L=Saint Louis, ST=Missouri, C=US
      /authorizers/accessPolicyProvider/property[@name="Node Identity 3"]: CN=nifi_server3.example.com, O=ExampleLLC, L=Saint Louis, ST=Missouri, C=US
      /authorizers/authorizer/identifier: managed-authorizer
      /authorizers/authorizer/class: org.apache.nifi.authorization.StandardManagedAuthorizer
      /authorizers/authorizer/property[@name="Access Policy Provider"]: file-access-policy-provider
    state_management:
      /stateManagement/cluster-provider/property[@name="Connect String"]: "{{ nifi_properties['nifi.zookeeper.connect.string'] }}"
    # Assuming nifi_server1 = 192.168.1.10, nifi_server2 = 192.168.1.11, nifi_server3 = 192.168.1.12
    # we have Ansible automatically set the myid file on each host to last octet of the node's IP address
    # and we set the 'X' of the zookeeper['server.X'] keys to those same numbers.
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
