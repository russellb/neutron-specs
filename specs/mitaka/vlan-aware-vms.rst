..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================
VLAN aware VMs
==========================================

Launchpad blueprint:

 https://blueprints.launchpad.net/neutron/+spec/vlan-aware-vms

This blueprint proposes how to incorporate VLAN aware VMs into
OpenStack. In this document a VLAN aware VM is a VM that sends and receives
VLAN tagged frames over its vNIC. Pros are better scalability and more
dynamic handling of Neutron networks connected to VMs.


Problem Description
===================

Currently VLANs in VMs are not integrated with Neutron. In certain OVS
based plugins VMs can only send untagged traffic. This is because OVS
currently does not support QinQ. Even if OVS did support QinQ we would need
something to integrate VM tagged traffic with rest of Neutron.

Use cases:

* There are old legacy applications that are VLAN aware. These applications
  can more easily be adapted to OpenStack.

* Some applications handle a large number (+100) of Neutron networks. It is
  more optimized to use VLANs instead of using one vNIC per network.

* VLANs are more dynamic. It's usually less complex to add/remove a VLAN
  than to add/remove a vNIC, from a VM perspective.

* Service VMs could use this to serve VMs from other Tenants if admin gives
  adequate permissions to the Service VMs.

* This proposal gives an explicit way to control VLAN restrictions on trunk
  ports. One can choose which VLANs can pass which port, thus restricting a
  VLAN to a subset of VMs. The approach requires an underlying mechanism
  to be VLAN-aware, not VLAN-agnostic.

* VLAN-aware mechanism can be used to expose particular VLANs from within
  the Neutron infrastructure to an external networks. The usecase like the
  one above with inter-VM VLAN restrictions.

* The existing hardware can effectively work with VLANs and QinQ, but QinQ
  still is not supported by Neutron, while the VLAN-transparent (or data
  agnostic) way can lead to issues with the hardware.

* A VM may be running many containers. Each container may have requirements
  to be connected to different Neutron networks. Assigning a VLAN id for
  each container is a nice way to handle this.

The proposal may look related in some sense to the VLAN-transparent
approach [1], but differs in that it doesn't provide a way to send opaque
data through the links, but instead be aware of the encapsulating, provided
by the VM (or an appliance inside a VM).


Proposed Change
===============

The proposal is to add a subport API extension.  The best way to think about
this is that Neutron needs a method to describe a logical topology such that you
have a regular VIF port (the parent) and that parent may also have child ports.
These child ports are used to describe logical entities that reside behind the
regular port.  Both the parent and child ports have all of the attributes that a
port has today and can be created on any Neutron network.  A VLAN ID is just one
possible encapsulation to use between the hypervisor and the guest to
differentiate packets targeted at the regular port vs. one of the child ports.

The first part of the API extension is the logical topology portion.  First, we
include a method to indicate that a port may have child ports.  This isn't
strictly necessary, but is included for convenience, both in the API and for
backend implementations.  This must be specified at port creation time and may
not be changed.

Note that all example command lines are for demonstrating the concepts.  The
exact syntax is subject to change.

::

    $ neutron port-create --subport:is_parent NETWORK

Note that if a given plugin does not implement support for the subport API
extension, it is completely safe to ignore this and implement the port as you
normally would.  The port handles untagged traffic and should behave as usual.
We could also update Neutron such that it rejects the request if the plugin does
not indicate that it supports the subport API extension.

The second part of the logical topology is the ability to specify that a port
has a parent.

::

    $ neutron port-create --subport:parent_id PARENT-PORT-ID NETWORK

Again, it's worth considering what happens if a plugin does not support this API
extension.  Nova (or whatever system is using Neutron) will never bind these
ports to a hypervisor.  That will only happen for the parent port.  If a plugin
doesn't support this API extension, these ports will never be bound and will
effectively be a NO-OP.  Of course, a better solution would be to just make
Neutron reject the port creation if the plugin does not indicate that it
supports the subport API extension.

In addition to the logical topology piece, we need to define the type of
encapsulation used between the hypervisor and VM for targeting packets to a
logical sub-port, as well as for determing the source logical port for a packet
arriving from the VM.  A VLAN ID is just one possible encapsulation.  There are
cases where other encapsulation types may be used.  For example, in the SFC
world there is talk of using several different encapsulation types as methods
for targeting logical service chain endpoints inside of a VM.  This API could
support modeling that.

A subport has both an encapsulation type, as well as some encapsulation info.
If the type is vlan, then the info is just an integer that represents a valid
VLAN ID.

::
    $ neutron port-create --subport:parent_id PARENT-PORT-ID \
    > --subport:encap_type vlan --subport:encap_info 101 NETWORK

For the purposes of this proposal, "vlan" is the only accepted encap_type at
this time.  Later proposals could define new encap types and associated
encapsulation info.

Just to reiterate, the encapsulation is a local encapsulation used between the
VM and hypervisor only.  It has nothing to do with how the Neutron network the
port is attached to is implemented.  It is only used to identify the source or
destination logical port over a single VIF.

Example logical model::

                          +---------+
                          | S1      |------------ N1
                          | VLAN 10 |
   +------+               +---------+
   |      |              /
   |      |+----------+ /
   |      ||          |/  +---------+
   |  VM  ||    P0    |---| S2      |------------ N2
   |      || Untagged |\  | VID: 20 |
   |      |+----------+ \ +---------+
   |      |     |        \
   +------+     |         +---------+
                |         | S3      |------------ N3
                N0        | VLAN 30 |
                          +---------+

* PO = Regular VIF port
* S1-S3 = Sub-ports
* N0-N3 = Networks

In the above example, a VM connects to 4 Neutron networks.  P0 is a regular port
that handles untagged traffic.  P0 has three subports, each using VLAN
encapsulation with a different VLAN ID.  Packets targeted at a subport will be
tagged with the appropriate VLAN ID before being sent to P0.  Packets received
on P0 tagged with a VLAN ID associated with a sub-port will be treated as if the
logical source was that subport and the packet will be sent to the Neutron
network that sub-port is attached to.

+---------+--------------+
|VM sends |Frame goes to |
+=========+==============+
|Untagged | N0           |
+---------+--------------+
|VID 10   | N1           |
+---------+--------------+
|VID 20   | N2           |
+---------+--------------+
|VID 30   | N3           |
+---------+--------------+

Constraints
-----------

* A subport must have exactly one parent, a port marked as being a parent.

* Every subport of a given parent should have unique VID among its
  siblings.  Otherwise, it's not possible to properly differentiate
  the source and destination logical port.

* The parent port handles "untagged" traffic. The parent will receive all the
  packets that do not match a sub-port, which is no different than how a regular
  port already behaves today.  This also means that every VM has a port for
  untagged traffic.  It doesn't necessarily have to be used though and could be
  attached to a dummy Neutron network if desired.  You could also have a
  security group set on the parent port to drop all incoming and outgoing
  traffic.  A future enhancement could include the ability to create a Neutron
  port not yet attached to a network, though it's unclear how valuable that
  actually is.

* When the parent port is bound, all the subports are marked as bound.
  This is handled by the Neutron plugin implementing subport support.

* A normal user can only connect subports to its own parent ports. Admin
  user can connect subports to parent ports with different owners.


Nova changes
------------

No changes are required in Nova as a parent port is a regular Neutron port.

Alternatives
------------

A previous version of this proposal included a new first class resource called a
"trunk port" instead of using a regular Neutron port as the parent.  The major
functional difference between the two is that in this proposal, the parent port
handles untagged traffic.  This is discussed in a bit more detail in the
"Constraints" section.  Some benefits of going with the existing port resources
instead of a new resource type include:

* It is my gut feeling that this proposal will result in a simpler
  implementation that requires less code, though that will have to be proven
  with code, which I'd like to help with to help move things along.
* No changes are required to any project that handles VIF port
  bindings, such as Nova or Kuryr, since a parent port is just a regular Neutron
  port resource.

One alternative is to extend the port so that it can be connected to
multiple networks directly. The main disadvantage with that is that it
would change the port in such way that it affected other services in
neutron that uses port information from ports connected to VMs. An example
of this is the DHCP agent. Another benefit of using solution with parent
ports and subports is that it probably is easier support service VMs that
connect to multiple Tenants. There will be one subport on each Tenant
network instead of one port connected to networks from different networks.

An option to associate the VID on the subport could be to associate it with
the network and let the tenants decide VID per network. A drawback with
this would be in the case a trunk port is used to connect to several
tenants. Different tenants could select the same VID for networks connected
to the service VM.

Another option could be to not have a separate VID associated with the
network but use the segmentation ID instead. This would limit this feature
to VLAN based Neutron networks. Also, if VIDs can't be chosen freely by the
users, more logic is needed in users to determine which VIDs to use.

Another alternative is to use a L2-gateway together with a true trunk
Neutron network (a Neutron network that can carry VLAN tagged
traffic).

Things to consider would be:
  * How to manage a VMs IP addresses for different VLANs on a trunk Neutron
    network. DHCP support for these IP addresses.
  * How to control the broadcast domains for the different VLANs. Possibly
    add support to select which VLANs a Neutron port is member of.
  * The most straight forward solution in plugins using OVS requires
    addition of QinQ support in OVS. Although VLAN in top of GRE and VXLAN
    type networks could probably be implemented using OF metadata.

Benefits would be that user do not have to manage each VLAN
separately. This means easier usage when trunking many VLANs between points
without caring about the content. Also less Neutron networks would be
consumed.

Another benefit is that L2 gateway is a
better solution when generic tranlation between VLAN, VXLAN etc. is exposed
to Tenant. See next section regarding adding this support to proposed
proposal. Different implementations of the L2 gateway could support
different things. To handle this a API to request/query support of L2
gateways could be added.

Current proposal is to always translate to/from VLAN for VM side. The
network_type for a neutron network could be VLAN, VXLAN or other and will
be translated to VLAN when send to VM. A more generic alternative could be
to be able to specify what to translate to/from. It would be possible for a
user to select how a network would be represented in a VM (VLAN, VXLAN or
other). This would be more complex solution and also require more
parameters, like IP addresses for VXLAN. All information required to setup
the specified network_type needs to exist in the API parameters.

Data Model Impact
-----------------

Neutron port:
  * subport:is_parent - Indicate that a port is a parent port
  * subport:parent_id - port id of parent.
  * subport:encap_type - Encapsulation type for subport traffic (vlan only for
    now)
  * subport:encap_info - For "vlan" encap_type, a integer that represents a
    valid VLAN ID.

REST API Impact
---------------

Subport extended attributes for port resource

+----------+-------+---------+---------+------------+---------------------+
|Attribute |Type   |Access   |Default  |Validation/ |Description          |
|Name      |       |         |Value    |Conversion  |                     |
+==========+=======+=========+=========+============+=====================+
|subport:  |bool   |CR, all  |false    |bool        |Indicate that a port |
|is_parent |       |         |         |            |may have child       |
|          |       |         |         |            |subports.            |
+----------+-------+---------+---------+------------+---------------------+
|subport:  |string |CR, all  |''       |uuid of     |ID of the parent     |
|parent_id |(UUID) |         |         |parent      |port that subport is |
|          |       |         |         |            |connected to.        |
+----------+-------+---------+---------+------------+---------------------+
|subport:  |integer|CR, all  |''       |uint or     |For encap_type of    |
|encap_info|       |         |         |''          |vlan, this is the    |
|          |       |         |         |            |VLAN id used for     |
|          |       |         |         |            |packets to and from  |
|          |       |         |         |            |this subport.        |
+----------+-------+---------+---------+------------+---------------------+
|subport:  |string |CR, all  |'vlan'   |enum (*)    |Encapsulation type   |
|encap_type|       |         |         |            |for packets to/from  |
|          |       |         |         |            |the subport as they  |
|          |       |         |         |            |pass through the     |
|          |       |         |         |            |parent port.         |
+----------+-------+---------+---------+------------+---------------------+

(*) As the blueprint is focused on VLAN tagging, only 'vlan' encap_type
is defined here. Any extension is a subject of further separate blueprints.

Security Impact
---------------

Only admin can create and attach subports for different tenants.

Notifications Impact
--------------------

None

Other End User Impact
---------------------

Tenants should be aware that OpenStack does nothing to enable VMs
to handle the tagged traffic, but just provides tagged packets. It
is totally up to the user to set VMs up properly.

Performance Impact
------------------

The performance of existing functionality should be unaffected.  The data path
for normal ports is unchanged.

In some cases, this change may improve performance.  Without this change,
connecting a VM to many Neutron networks required a VIF per network.  With this
change, you could connect to 1000 Neutron networks with very little overhead vs.
having to attach 1000 virtual interfaces to your VM before.

IPv6 Impact
-----------

None.  Both parent ports and subports have all of the same attributes as Neutron
ports do today, including IPv6 addresses if desired.

Other Deployer Impact
---------------------

None

Developer Impact
----------------

* Extented attributes on the Neutron port are to be used.
* Requires modifications to Neutron plugins to support this model
* Requires development of new tests.

Community Impact
----------------

Implementation
==============

Assignee(s)
-----------

Kevin Benton
Peter V. Saveliev
Russell Bryant

Work Items
----------

* API extension and DB schema updates
* Unit tests for API+DB changes
* Tempest tests for creating port topology
* Tempest scenario test(s) for doing functional validation
* Neutron Plugin support.
  * networking-ovn (OVN supports this model already)
  * ml2+ovs

Dependencies
============

Testing
=======

Tempest and functional tests will be created.

Tempest Tests
-------------

Tempest tests to be implemented:

* Create parent ports
* Create subports
* Bind parent ports
* Delete subports
* Delete parent ports

Functional Tests
----------------

Tests to be implemented:

* Boot VM with one parent port and no children.
  Verify connectivity.
* Boot VM with one parent port and multiple subports.
  Verify connectivity to each logical port.
* Boot VM with multiple parent ports and subports and verify connectivity.
* Add subport to running VM with a parent port.
* Remove subport from running VM with parent port.
* Delete VM with parent port including subports.

API Tests
---------

Tests to be implemented:

* Check that subport only can be connected to a parent port.
* Check that an invalid encap_type is rejected
* Check that an invalid encap_info is rejected


Documentation Impact
====================

The use of parent ports and subports should be documented as a way to create a
logical multi-port topology using a single VIF on a VM.

Possible scenarios for usecases should be provided with
CLI examples.

User Documentation
------------------

Update networking API reference.
Update admin guide.

Developer Documentation
-----------------------

The trunk port resource description, subports behaviour, API reference.

References
==========

None

