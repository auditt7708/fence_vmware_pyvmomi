# fence_vmware_pyvmomi
``fence_vmware_pyvmomi`` is a STONITH plugin for fencing virtual machines on ESX/ESXi hosts and vCenter Server via pyVmomi (*[VMware vSphere API Python Bindings](https://github.com/vmware/pyvmomi)*).

# Why?
Yes, I know - there already some STONITH plugins for VMware - such as ``external/vmware`` and ``external/fence_vmware_soap``. While the first one looks pretty outdated to me, the second alternative requires the [VMware vSphere Perl SDK](https://my.vmware.com/de/web/vmware/details?downloadGroup=VS-PERL-SDK65&productId=614). Unfortunately, this SDK is only supported for some selected Linux distributions (*such as Enterprise Linux and some Ubuntu versions*). On the other hand, the ([VMware vSphere API Python Bindings](https://github.com/vmware/pyvmomi)) are easier to handle and work on a broader range of Linux distributions.

# Requirements
To use this plugin, you will need:
- [pyVmomi](https://github.com/vmware/pyvmomi) Python bindings
  - **Version 6.0** or higher! Use [PIP](https://pypi.python.org) if your Linux distribution's package is outdated! (*otherwise you will get SSL errors*)
- Python modules ``requests``, ``socket``, ``logging`` and ``urlparse``
- vCenter access

## vCenter permissions
In order to control a particular VM's power state, you will need the following permissions to the corresponding object:
- Virtual machine
  - Interaction
    - Power on
    - Power off

For the datacenter object, you will need read access - otherwise the user won't be able to "*find*" the VM.

# Usage
## Manually
The script utilizes shell variables to assign connection details:

| Variable | Description |
|:---------|:------------|
| ``debug`` | Set to '1' to enable debugging |
| ``hostname`` | vCenter Server / ESXi hostname |
| ``port`` | SSL port |
| ``username`` | Service username |
| ``password`` | Appropriate password |
| ``vm_name`` | VM name |

### Examples
Retrieve VM power state:
```
$ hostname="vcenter.localdomain.loc" port="443" username="vsphere.local\giertz" password="chad" vm_name="giertz" debug=1 ./fence_vmware_pyvmomi.py status
poweredOff
```

Power-off VM:
```
$ hostname="vcenter.localdomain.loc" port="443" username="vsphere.local\giertz" password="chad" vm_name="giertz" debug=1 ./fence_vmware_pyvmomi.py off
```

Power-on VM:
```
$ hostname="vcenter.localdomain.loc" port="443" username="vsphere.local\giertz" password="chad" vm_name="giertz" debug=1 ./fence_vmware_pyvmomi.py on
```

## Via ``stonith``
Move ``fence_vmware_pyvmomi`` to ``/usr/lib/stonith/plugins/external``.

### Examples
Retrieve VM power state:
```
# stonith -t external/fence_vmware_pyvmomi hostname=vcenter.localdomain.loc username="vsphere.local\giertz" password="chad" vm_name="pinkepank" -S
info: external_run_cmd: '/usr/lib/stonith/plugins/external/fence_vmware_pyvmomi status' output: poweredOff

info: external/fence_vmware_pyvmomi device OK.
```

Power-off VM:
```
# stonith -t external/fence_vmware_pyvmomi hostname=vcenter.localdomain.loc username="vsphere.local\giertz" password="chad" vm_name="pinkepank" -T off pinkepank
```

Power-on VM:
```
# stonith -t external/fence_vmware_pyvmomi hostname=vcenter.localdomain.loc username="vsphere.local\giertz" password="chad" vm_name="pinkepank" -T on pinkepank
```

## Via pacemaker
The first step is to create primitives for your cluster nodes withing the ``crm`` shell:
```
primitive fence_vcenter_nodeA stonith:external/fence_vmware_pyvmomi \
    op monitor interval=60 timeout=120 \
    params username="vsphere.local\giertz" password="chad" hostname=vcenter.localdomain.loc port=443 vm_name=nodeA
primitive fence_vcenter_nodeB stonith:external/fence_vmware_pyvmomi \
    op monitor interval=60 timeout=120 \
    params username="vsphere.local\giertz" password="chad" hostname=vcenter.localdomain.loc port=443 vm_name=nodeB
...
```

The next step is to create locations to ensure that cluster nodes are fencing each other:
```
location loc_stonith-deb9a fence_vcenter_nodeA \
	-inf: nodeA.localdomain.loc
location loc_stonith-deb9b fence_vcenter_nodeB \
   	-inf: nodeB.localdomain.loc
```

Finally, enable STONITH and - if required - disable the ignore quorum policy:
```
property cib-bootstrap-options: \
...
stonith-enabled=true \
stonith-action=poweroff \
no-quorum-policy=ignore \
...
```

Try fencing a particular node:
```
# crm node fence nodeA.localdomain.loc
```
