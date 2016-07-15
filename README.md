Vxlan Role for EOS
==================

The arista.eos-vxlan role creates an abstraction for common EOS
Vxlan configuration. This means that you do not need to write any ansible tasks.
Simply create an object that matches the requirements below
and this role will ingest that object and perform the necessary configuration.

This role is used to configure logical Vxlan interfaces, Vlan to VNI mapping,
and the VTEP global/vlan flood list. It will **NOT** create the underlying
Vlans, so it is recommended to use arista.eos-bridging to do that.  These two
roles work in tandem nicely by analyzing the same ``vlans`` object to drive
their respective tasks (more info below).

Installation
------------

```
ansible-galaxy install arista.eos-vxlan
```


Requirements
------------

Requires an SSH connection for connectivity to your Arista device. You can use
any of the built-in eos connection variables, or the convenience ``provider``
dictionary.

Role Variables
--------------

The tasks in this role are driven by the ``vxlan``, ``vteps``,
``vlans``, and ``eos_purge_vxlan`` objects described below:

**vxlan** (dictionary) contains the following keys:

|              Key | Type                      | Notes                                                                                                                                                                                                                                                 |
|-----------------:|---------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
|             name | string (required)         | The unique interface identifier name. The interface name must use the full interface name (no abbreviated names). Note: The name parameter only accepts Vxlan1 as the identifier                                                                      |
|      description | string                    | Configures a one line ASCII description for the interface. The EOS default value for description is None                                                                                                                                               |
|           enable | boolean: true*, false     | Configures the administrative state for the interface. Setting the value to true will administratively enable the interface and setting the value to false will administratively disable the interface. The EOS default value for enable is true         |
| source_interface | string                    | Configures the vxlan source-interface value which directs the interface to use the specified source interface address to source messages from. The configured value must be a Loopback interface. The EOS default value for source interface is None. |
|  multicast_group | string                    | Configures the vxlan multicast-group address used for flooding traffic between VTEPs. This value must be a valid multicast address in the range of 224/8. The EOS default value for vxlan multicast-group is None.                                    |
|         udp_port | string                    | Configures the vxlan udp-port value used to terminate multicast messages between VTEPs. This value must be an integer in the range of 1024 to 65535. The EOS default value for vxlan udp-port is 4789.                                                |
|            state | choices: present*, absent | Set the state for the Vxlan interface configuration.                                                                                                                                                                                                  |

```
Note: If the vxlan object is not defined or has state set to absent, then
the vteps, vlans, and eos_purge_vxlan objects are ignored, and the related
tasks from this role are skipped.
```

**vteps** (list) each entry contains the following keys:

|   Key | Type                      | Notes                                                                                                                                                                            |
|------:|---------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
|  name | string                    | The unique interface identifier name. The interface name must use the full interface name (no abbreviated names). Note: Defaults to Vxlan1, and any other name will result in error. |
|  vtep | string                    | Specifies the remote endpoint IP address to add to the global VTEP flood list. Valid values for the vtep parameter are unicast IPv4 addresses. Multiple addresses may be specified as a space separated list, or as a yaml format list (see examples). |
|  vlan | string                    | Specifies the VLAN ID to associate the VTEP with. If the VLAN argument is not used, the the VTEP is configured on the global flood list.                                         |
| state | choices: present*, absent | Set the state for the Vxlan interface configuration.                                                                                                                             |
| reset | choices: true, false*     | If True, remove all existing vtep entries from the specified vlan before performing adding or removing named entries. See clarification note below and examples.                 |

```
Note: clarification of state/reset combinations
  present/true - will replace any existing vtep entries with the specified vtep entries
  present/false - will add the specified entries to any existing enties
  absent/true - will remove all vtep entries for the specified vlan, regardless of what vtep value is given
  absent/false - will remove only the specified vtep entries from the named vlan
```

**vlans** (list) each entry contains the following keys:

|    Key | Type                      | Notes                                                                                                                                                                                                                                                                  |
|-------:|---------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| vlanid | string (required)         | Specifies the VLAN ID that is associated with the Vxlan interface for managing the VLAN to VNI mapping. Valid values for the vlan parameter are in the range of 1 to 4094.                                                                                              |
|    vni | string                    | Specifies the VNI value to associate with the Vxlan interface for managing the VLAN to VNI mapping. This value is only necessary when configuring the mapping with a state of present (default). Valid values for the vni parameter are in the range of 1 to 16777215. |
|  state | choices: present*, absent | Set the state for the VNI mapping interface configuration.                                                                                                                                                                                                             |

```
Note: The vlans list is the same list that is used in the arista.eos-bridging role.
This means you can mix vxlan and non-vxlan vlans in the same list object. If the
vni key is set, this role will be used to set the mapping. If vni is not included,
the vlan will be skipped in this role.
```

**eos_purge_vxlan**

|             Key |          Type         | Notes                                                                                             |
|----------------:|:---------------------:|---------------------------------------------------------------------------------------------------|
| eos_purge_vxlan | boolean: true, false* | Enable or disable purging of Vlan to VNI mappings that are not defined in the vxlan_vlans object. |

```
Note: Asterisk (*) denotes the default value if none specified
```

Configuration Variables
-----------------------

|                     Key | Choices      | Description                              |
| ----------------------: | ------------ | ---------------------------------------- |
| eos_save_running_config | true*, false | Specifies whether to write any changes to the running-config resulting from the role execution to memory, copying the configuration to the startup-config. |

```
Note: Asterisk (*) denotes the default value if none specified
```

Connection Variables
--------------------

Ansible EOS roles require the following connection information to establish
communication with the nodes in your inventory. This information can exist in
the Ansible group_vars or host_vars directories, or in the playbook itself.

|         Key | Required | Choices    | Description                              |
| ----------: | -------- | ---------- | ---------------------------------------- |
|        host | yes      |            | Specifies the DNS host name or address for connecting to the remote device over the specified *transport*. The value of *host* is used as the destination address for the transport. |
|        port | no       |            | Specifies the port to use when building the connection to the remote device. This value applies to either acceptable value of *transport*. The port value will default to the appropriate transport common port if none is provided in the task (cli=22, http=80, https=443). |
|    username | no       |            | Configures the usename to use to authenticate the connection to the remote device.  The value of *username* is used to authenticate either the CLI login or the eAPI authentication depending on which *transport* is used. If the value is not specified in the task, the value of environment variable ANSIBLE_NET_USERNAME will be used instead. |
|    password | no       |            | Specifies the password to use to authenticate the connection to the remote device. This is a common argument used for either acceptable value of *transport*. If the value is not specified in the task, the value of environment variable ANSIBLE_NET_PASSWORD will be used instead. |
| ssh_keyfile | no       |            | Specifies the SSH keyfile to use to authenticate the connection to the remote device. This argument is only used when *transport=cli*. If the value is not specified in the task, the value of environment variable ANSIBLE_NET_SSH_KEYFILE will be used instead. |
|   authorize | no       | yes, no*   | Instructs the module to enter priviledged mode on the remote device before sending any commands. If not specified, the device will attempt to excecute all commands in non-priviledged mode. If the value is not specified in the task, the value of environment variable ANSIBLE_NET_AUTHORIZE will be used instead. |
|   auth_pass | no       |            | Specifies the password to use if required to enter privileged mode on the remote device.  If *authorize=no*, then this argument does nothing. If the value is not specified in the task, the value of environment variable ANSIBLE_NET_AUTH_PASS will be used instead. |
|   transport | yes      | cli*, eapi | Configures the transport connection to use when connecting to the remote device. The *transport* argument supports connectivity to the device over cli (ssh) or eapi. |
|     use_ssl | no       | yes*, no   | Configures the transport to use SSL if set to true only when *transport=eapi*.  If *transport=cli*, this value is ignored. |
|    provider | no       |            | Convience method that allows all the above connection arguments to be passed as a dict object. All constraints (required, choices, etc) must be met either by individual arguments or values in this dict. |

```
Note: Asterisk (*) denotes the default value if none specified
```

Ansible Variables
-----------------

|    Key | Choices      | Description                              |
| -----: | ------------ | ---------------------------------------- |
| no_log | true, false* | Prevents module arguments and output from being logged during the playbook execution. By default, no_log is set to true for tasks that gather and save EOS configuration information to reduce output size. Set to true to prevent all output other than task results. |

```
Note: Asterisk (*) denotes the default value if none specified
```


Dependencies
------------

The eos-vxlan role is built on modules included in the core Ansible code.
These modules were added in ansible version 2.1

- Ansible 2.1.0
- arista.eos-bridging (not strictly required, but recommended to create the vlans)

Example Playbook
----------------

The following example will use the arista.eos-vxlan role to configure
common Vxlan settings. We'll create a
``hosts`` file with our switch, then a corresponding ``host_vars`` file and
then a simple playbook which only references the eos-vxlan role.
By including the role, we automatically get access to all of the tasks
to configure these EOS features. What's nice about this is that if you have a
host without any corresponding configuration, the tasks will be skipped
without any issue.


Sample hosts file:

    [leafs]
    leaf1.example.com

Sample host_vars/leaf1.example.com

    provider:
      host: "{{ inventory_hostname }}"
      username: admin
      password: admin
      use_ssl: no
      authorize: yes
      transport: cli

    vxlan:
        name: Vxlan1
        source_interface: Loopback0
        multicast_group: 239.10.10.10

    vteps:
      # add a global vtep entry
      - vtep: 4.4.4.4
      # add vlan specific entries
      - vtep: 1.1.1.1 2.2.2.2
        vlan: 101
      # replace vlan specific entries
      - vtep:
        - 9.9.9.9
        - 8.8.8.8
        vlan: 102
        reset: true
      # remove all vlan vtep entries (vtep is ignored)
      - vtep: 1.2.3.4
        vlan: 104
        state: absent
        reset: true
      # remove all vlan vtep entries (no vtep specified)
      - vlan: 105
        state: absent
        reset: true
      # remove specific vlan vtep entries only
      - vtep: 6.6.6.6 2.2.2.2
        vlan: 106
        state: absent
        reset: false

    vlans:
      - vlanid: 1
        name: default
      - vlanid: 100
        name: VXLAN_VLAN_100
        vni: 1000
      - vlanid: 101
        name: VXLAN_VLAN_101
        vni: 101
      - vlanid: 102
        name: VLAN_102
      - vlanid: 103
        name: VXLAN_VLAN_103
        vni: 103
      - vlanid: 104
        name: VXLAN_VLAN_104
        vni: 104
      - vlanid: 105
        name: VXLAN_VLAN_105
        vni: 105
      - vlanid: 106
        name: VXLAN_VLAN_106
        vni: 106
      - vlanid: 107
        name: VXLAN_VLAN_107


A simple playbook, leaf.yml. Here we use the bridging role to create the Vlans
themselves and then use the vxlan role to do the mapping.

    - hosts: leafs
      roles:
        - arista.eos-bridging
        - arista.eos-vxlan

Then run with:

    ansible-playbook -i hosts leaf.yml
  ​    


  Developer Information
  ------------

  Development contributions are welcome. Please see *Arista Roles for Ansible - Development Guidelines* ([test/arista-ansible-role-test/README](test/arista-ansible-role-test/README.md)) for additional information, including how to develop and run test cases for role development.




  License
  -------

  Copyright (c) 2015, Arista Networks EOS+
  All rights reserved.

  Redistribution and use in source and binary forms, with or without
  modification, are permitted provided that the following conditions are met:

  * Redistributions of source code must retain the above copyright notice, this
    list of conditions and the following disclaimer.

  * Redistributions in binary form must reproduce the above copyright notice,
    this list of conditions and the following disclaimer in the documentation
    and/or other materials provided with the distribution.

  * Neither the name of Arista nor the names of its
    contributors may be used to endorse or promote products derived from
    this software without specific prior written permission.

  THIS SOFTWARE IS PROVIDED BY THE COPYRIGHT HOLDERS AND CONTRIBUTORS "AS IS"
  AND ANY EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT LIMITED TO, THE
  IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS FOR A PARTICULAR PURPOSE ARE
  DISCLAIMED. IN NO EVENT SHALL THE COPYRIGHT HOLDER OR CONTRIBUTORS BE LIABLE
  FOR ANY DIRECT, INDIRECT, INCIDENTAL, SPECIAL, EXEMPLARY, OR CONSEQUENTIAL
  DAMAGES (INCLUDING, BUT NOT LIMITED TO, PROCUREMENT OF SUBSTITUTE GOODS OR
  SERVICES; LOSS OF USE, DATA, OR PROFITS; OR BUSINESS INTERRUPTION) HOWEVER
  CAUSED AND ON ANY THEORY OF LIABILITY, WHETHER IN CONTRACT, STRICT LIABILITY,
  OR TORT (INCLUDING NEGLIGENCE OR OTHERWISE) ARISING IN ANY WAY OUT OF THE USE
  OF THIS SOFTWARE, EVEN IF ADVISED OF THE POSSIBILITY OF SUCH DAMAGE.


  Author Information
  ------------------

  Please raise any issues using our GitHub repo or email us at ansible-dev@arista.com
