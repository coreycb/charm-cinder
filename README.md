Overview
--------

This charm provides the Cinder volume service for OpenStack.  It is intended to
be used alongside the other OpenStack components, starting with the Folsom
release.

Cinder is made up of 3 separate services: an API service, a scheduler and a
volume service.  This charm allows them to be deployed in different
combination, depending on user preference and requirements.

This charm was developed to support deploying Folsom on both
Ubuntu Quantal and Ubuntu Precise.  Since Cinder is only available for
Ubuntu 12.04 via the Ubuntu Cloud Archive, deploying this charm to a
Precise machine will by default install Cinder and its dependencies from
the Cloud Archive.

Usage
-----

Cinder may be deployed in a number of ways.  This charm focuses on 3 main
configurations.  All require the existence of the other core OpenStack
services deployed via Juju charms, specifically: mysql, rabbitmq-server,
keystone and nova-cloud-controller.  The following assumes these services
have already been deployed.

Basic, all-in-one using local storage and iSCSI
===============================================

The api server, scheduler and volume service are all deployed into the same
unit.  Local storage will be initialized as a LVM phsyical device, and a volume
group initialized.  Instance volumes will be created locally as logical volumes
and exported to instances via iSCSI.  This is ideal for small-scale deployments
or testing:

    cat >cinder.cfg <<END
    cinder:
        block-device: sdc
        overwrite: true
    END
    juju deploy --config=cinder.cfg cinder
    juju add-relation cinder keystone
    juju add-relation cinder mysql
    juju add-relation cinder rabbitmq-server
    juju add-relation cinder nova-cloud-controller

Separate volume units for scale out, using local storage and iSCSI
==================================================================

Separating the volume service from the API service allows the storage pool
to easily scale without the added complexity that accompanies load-balancing
the API server.  When we've exhausted local storage on volume server, we can
simply add-unit to expand our capacity.  Future requests to allocate volumes
will be distributed across the pool of volume servers according to the
availability of storage space.

    cat >cinder.cfg <<END
    cinder-api:
        enabled-services: api, scheduler
    cinder-volume:
        enabled-services: volume
        block-device: sdc
        overwrite: true
    END
    juju deploy --config=cinder.cfg cinder cinder-api
    juju deploy --config=cinder.cfg cinder cinder-volume
    juju add-relation cinder-api mysql
    juju add-relation cinder-api rabbitmq-server
    juju add-relation cinder-api keystone
    juju add-relation cinder-api nova-cloud-controller
    juju add-relation cinder-volume mysql
    juju add-relation cinder-volume rabbitmq-server

    # When more storage is needed, simply add more volume servers.
    juju add-unit cinder-volume

All-in-one using Ceph-backed RBD volumes
========================================

All 3 services can be deployed to the same unit, but instead of relying
on local storage to back volumes an external Ceph cluster is used.  This
allows scalability and redundancy needs to be satisified and Cinder's RBD
driver used to create, export and connect volumes to instances.  This assumes
a functioning Ceph cluster has already been deployed using the official Ceph
charm and a relation exists between the Ceph service and the nova-compute
service.

    cat >cinder.cfg <<END
    cinder:
        block-device: None
    END
    juju deploy --config=cinder.cfg cinder
    juju add-relation cinder ceph
    juju add-relation cinder keystone
    juju add-relation cinder mysql
    juju add-relation cinder rabbitmq-server
    juju add-relation cinder nova-cloud-controller


Configuration
-------------

The default value for most config options should work for most deployments.

Users should be aware of three options, in particular:

openstack-origin:  Allows Cinder to be installed from a specific apt repository.
                   See config.yaml for a list of supported sources.

block-device:  When using local storage, a block device should be specified to
               back a LVM volume group.  It's important this device exists on
               all nodes that the service may be deployed to.

overwrite:  Whether or not to wipe local storage that of data that may prevent
            it from being initialized as a LVM phsyical device.  This includes
            filesystems and partition tables.  *CAUTION*

enabled-services:  Can be used to separate cinder services between service
                   service units (see previous section)

Deploying from source
---------------------

The minimal openstack-origin-git config required to deploy from source is:

  openstack-origin-git:
      "{'cinder':
           {'repository': 'git://git.openstack.org/openstack/cinder.git',
            'branch': 'stable/icehouse'}}"

If you specify a 'requirements' repository, it will be used to update the
requirements.txt files of all other git repos that it applies to, before
they are installed:

  openstack-origin-git:
      "{'requirements':
           {'repository': 'git://git.openstack.org/openstack/requirements.git',
            'branch': 'master'},
        'cinder':
           {'repository': 'git://git.openstack.org/openstack/cinder.git',
            'branch': 'master'}}"

Note that there are only two key values the charm knows about for the outermost
dictionary: 'cinder' and 'requirements'. These repositories must correspond to
these keys. If the requirements repository is specified, it will be installed
first. The cinder repository is always installed last.  All other repostories
will be installed in between.

NOTE(coreycb): The following is temporary to keep track of the full list of
current tip repos (may not be up-to-date).

  openstack-origin-git:
      "{'requirements':
           {'repository': 'git://git.openstack.org/openstack/requirements.git',
            'branch': 'master'},
        'keystonemiddleware:
           {'repository': 'git://git.openstack.org/openstack/keystonemiddleware.git',
            'branch: 'master'},
        'oslo-concurrency':
           {'repository': 'git://git.openstack.org/openstack/oslo.concurrency.git',
            'branch: 'master'},
        'oslo-config':
           {'repository': 'git://git.openstack.org/openstack/oslo.config.git',
            'branch: 'master'},
        'oslo-context':
           {'repository': 'git://git.openstack.org/openstack/oslo.context.git',
            'branch: 'master'},
        'oslo-db':
           {'repository': 'git://git.openstack.org/openstack/oslo.db.git',
            'branch: 'master'},
        'oslo-i18n':
           {'repository': 'git://git.openstack.org/openstack/oslo.i18n.git',
            'branch: 'master'},
        'oslo-messaging':
           {'repository': 'git://git.openstack.org/openstack/oslo.messaging.git',
            'branch: 'master'},
        'oslo-rootwrap':
           {'repository': 'git://git.openstack.org/openstack/oslo.rootwrap.git',
            'branch: 'master'},
        'oslo-serialization':
           {'repository': 'git://git.openstack.org/openstack/oslo.serialization.git',
            'branch: 'master'},
        'oslo-utils':
           {'repository': 'git://git.openstack.org/openstack/oslo.utils.git',
            'branch: 'master'},
        'oslo-vmware':
           {'repository': 'git://git.openstack.org/openstack/oslo.vmware.git',
            'branch: 'master'},
        'osprofiler':
           {'repository': 'git://git.openstack.org/stackforge/osprofiler.git',
            'branch: 'master'},
        'pbr':
           {'repository': 'git://git.openstack.org/openstack-dev/pbr.git',
            'branch: 'master'},
        'python-barbicanclient':
           {'repository': 'git://git.openstack.org/openstack/python-barbicanclient.git',
            'branch: 'master'},
        'python-glanceclient':
           {'repository': 'git://git.openstack.org/openstack/python-glanceclient.git',
            'branch: 'master'},
        'python-novaclient':
           {'repository': 'git://git.openstack.org/openstack/python-novaclient.git',
            'branch: 'master'},
        'python-swiftclient':
           {'repository': 'git://git.openstack.org/openstack/python-swiftclient.git',
            'branch: 'master'},
        'stevedore':
           {'repository': 'git://git.openstack.org/openstack/stevedore.git',
            'branch: 'master'},
        'sqlalchemy-migrate':
           {'repository': 'git://git.openstack.org/stackforge/sqlalchemy-migrate.git',
            'branch: 'master'},
        'taskflow':
           {'repository': 'git://git.openstack.org/openstack/taskflow.git',
            'branch: 'master'},
        'cinder':
           {'repository': 'git://git.openstack.org/openstack/cinder.git',
            'branch': 'master'}}"
