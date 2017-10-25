..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

===========================================
Support address-set in security-group-rules
===========================================

https://bugs.launchpad.net/neutron/+bug/1649417

This spec describes a way to support specifying a set of IP addresses in
security group rules.


Problem Description
===================

There are use cases that a set of IP addresses needs to be specified in
security group rules as remote IP addresses that access is allowed. These
IPs may or may not be used by any Neutron ports. A typical example is to allow
access from an application that is deployed in legacy or external systems that
are not managed by Neutron to access certain Neutron ports. There are 2 ways to
achieve this with existing Neutron API:

* Specify each individual IP in separate rules with the remote_ip_prefix field.
  There is a efficiency/scale problem when the number of IPs in the set gets too
  big. This will results in large number of rules, and also redundant data in each
  rule, since the only difference between those rules is the IP address.

* Create dummy ports with allowed-address-pairs, or with fixed-ips, and add the
  ports to a security group, and specify the security group as remote-group-id
  in the security group rule. This would work, but it is more of a workaround, and
  introduces cost of unnecessary port related operations.

Proposed Change
===============

The proposed change is to add support for address set at API level. Two new types
of resources need to be added as API extension:

* Address Sets - the container of a set of addresses
* Addresses - sub-resource of Address Sets, which can be in the format of individual
  IP addresses or IP prefixes. Both format of addresses can coexist in same address
  set.

A new optional attribute will be added to Security Group Rule resource:

* remote-address-set

.. Note::
   The attribute remote-address-set and remote-ip-prefix must not be specified
   in the same rule, to avoid confusion.

With the proposed API change, the work flow to support above use case will be:

#. Create an Address Set

#. Add Addresses in the Address Set

#. Create security group rule to allow access to/from the Address Set

Implementation of the API extension is just DB operations for address set data model
corresponding to the API.

Changes to CLI
--------------

Both OpenStack CLI and Neutron CLI shall be changed to:

* Provide new CLI commands to create/delete/show address sets, add/remove addresses
  to/from address sets
* Update security-group-rule related CLI to support the new attribute:
  remote-address-set

Changes to Callbacks
--------------------

To support the address set feature by plugins, we need to add ADDRESS_SET
as a new resource type, so that plugins can get notified for the address
set changes.

As to plugins, the change is backward compatible, because the "remote_address_set"
attribute is optional. To support the feature, the actual changes depends
on the security group rule enforcement mechanism behind.

Changes to RPCs
---------------

New interfaces shall be added to security group RPC handlers for address
set operations.

Changes to agent based firewalls
--------------------------------

The FirewallDriver class shall add new interface to support address set
create/update/delete operations.

Implementation of firewall drivers shall implement the new interfaces and
handle the "remote_address_set" attribute in security group rules.

For example, iptables_firewall can implement the address set using ipset,
while openvswitch_firewall can implement it by expanding to OpenFlow rules
accordingly.

Changes to plugins/ML2 drivers
------------------------------

For existing plugins/ML2 drivers, since the "remote_address_set" attribute is
optional, they can still work without having to make any change. If the plugin/ML2
driver intend to support the address set feature, the actual implementation
depends on the security group rule enforcement mechanism behind.

Take networking-ovn [1]_ as an example, the OVN backend provides direct support
for address set abstraction. The implementation should be straightforward, mapping
CRUD operations of address set directly to OVN-NB DB operations, translating
"remote_address_set" attribute in security group rules to address set in match
conditions of OVN ACLs.


References
==========

.. [1] https://docs.openstack.org/networking-ovn/latest/