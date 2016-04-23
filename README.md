Openstack Austin 2016 - Astara Workshop
===================

Assumptions
------------

You have a fully operating Openstack environment with, at least: Nova, Keystone, Glance, and Neutron
The OpenStack environment has been verified to be working, and VM instances can be deployed.


Architecture
------------

Openstack Controller with 3 NIC - Management, Tunnel, and External
Openstack Computer with 3 NIC - Management, Tunnel, and External


Installation
------------

All configuration is to be performed on the controller node.

1. Verify Operational Status of Openstack Deploymnet::

Source the openstack demouser rc file::

    % source /root/userrc

Create neutron resources - networks and router::

    % neutron net-create demo-net
    % neutron subnet-create --name demo-subnet demo-net 10.2.0.0/24
    % neutron router-create demo-router
    % neutron router-interface-add demo-router demo-subnet
