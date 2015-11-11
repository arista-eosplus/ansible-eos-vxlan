Vxlan Role for EOS
==================

The arista.eos-vxlan role creates an abstraction for common EOS
Vxlan configuration. This means that you do not need to write any ansible tasks.
Simply create an object that matches the requirements below
and this role will ingest that object and perform the necessary configuration.

This role is used to configure logical Vxlan interfaces, Vlan to VNI mapping,
and the VTEP global flood list. It will **NOT** create the underlying Vlans,
so it is recommended to use arista.eos-bridging to do that.  These two roles
work in tandem nicely by analyzing the same ``vlans`` object to drive
their respective tasks (more info below).

Installation
------------

```
ansible-galaxy install arista.eos-vxlan
```


Requirements
------------

Requires the arista.eos role.  If you have not worked with the arista.eos role,
consider following the [Quickstart][quickstart] guide.

Role Variables
--------------

The tasks in this role are driven by the ``vxlan``, ``vteps``,
``vxlan_vlans``, and ``eos_purge_vxlan`` objects described below:

**vxlan** (dictionary) contains the following keys:

|              Key | Type                      | Notes                                                                                                                                                                                                                                                 |
|-----------------:|---------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
|             name | string (required)         | The unique interface identifier name. The interface name must use the full interface name (no abbreviated names). Note: The name parameter only accepts Vxlan1 as the identifier                                                                      |
|      description | string                    | Configures a one lne ASCII description for the interface. The EOS default value for description is None                                                                                                                                               |
|           enable | boolean: true*, false     | Configures the administrative state for the interface. Setting the value to true will adminstrative enable the interface and setting the value to false will administratively disable the interface. The EOS default value for enable is true         |
| source_interface | string                    | Configures the vxlan source-interface value which directs the interface to use the specified source interface address to source messages from. The configured value must be a Loopback interface. The EOS default value for source interface is None. |
|  multicast_group | string                    | Configures the vxlan multicast-group address used for flooding traffic between VTEPs. This value must be a valid multicast address in the range of 224/8. The EOS default value for vxlan multicast-group is None.                                    |
|         udp_port | string                    | Configures the vxlan udp-port value used to terminate mutlicast messages between VTEPs. This value must be an integer in the range of 1024 to 65535. The EOS default value for vxlan udp-port is 4789.                                                |
|            state | choices: present*, absent | Set the state for the Vxlan interface configuration.                                                                                                                                                                                                  |

**vteps** (list) each entry contains the following keys:

|   Key | Type                      | Notes                                                                                                                                                                            |
|------:|---------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
|  name | string (required)         | The unique interface identifier name. The interface name must use the full interface name (no abbreviated names). Note: The name parameter only accepts Vxlan1 as the identifier |
|  vtep | string                    | Specifies the remote endpoint IP address to add to the global VTEP flood list. Valid values for the vtep parameter are unicast IPv4 addresses                                    |
|  vlan | string                    | Specifies the VLAN ID to associate the VTEP with. If the VLAN argument is not used, the the VTEP is confgured on the global flood list.                                          |
| state | choices: present*, absent | Set the state for the Vxlan interface configuration.                                                                                                                             |

**vlans** (list) each entry contains the following keys:

|   Key | Type                      | Notes                                                                                                                                                                                                                                                                  |
|------:|---------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
|  name | string (required)         | The unique interface identifier name. The interface name must use the full interface name (no abbreviated names). Note: The name parameter only accepts Vxlan1 as the identifier                                                                                       |
|   vni | string                    | Specifies the VNI value to assoicate with the Vxlan interface for managing the VLAN to VNI mapping. This value is only necessary when configuring the mapping with a state of present (default). Valie values for the vni parameter are in the range of 1 to 16777215. |
|  vlan | string                    | Specifies the VLAN ID that is assocated with the Vxlan interface for managing the VLAN to VNI mapping. Valid values for the vlan parameter are in the range of 1 to 4094.                                                                                              |
| state | choices: present*, absent | Set the state for the VNI mapping interface configuration.                                                                                                                                                                                                             |

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


Dependencies
------------

The eos-vxlan role utilizes modules distributed within the
arista.eos role.

- arista.eos version 1.2.0
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

    vxlan:
        name: Vxlan1
        source_interface: Loopback0
        multicast_group: 239.10.10.10

    vteps:
      - vtep: 4.4.4.4

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

[quickstart]: http://ansible-eos.readthedocs.org/en/latest/quickstart.html
