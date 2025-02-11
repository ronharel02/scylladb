.. |SCYLLA_NAME| replace:: ScyllaDB

.. |SRC_VERSION| replace:: 5.2
.. |NEW_VERSION| replace:: 5.4

.. |DEBIAN_SRC_REPO| replace:: Debian
.. _DEBIAN_SRC_REPO: https://www.scylladb.com/download/?platform=debian-10&version=scylla-5.2

.. |UBUNTU_SRC_REPO| replace:: Ubuntu
.. _UBUNTU_SRC_REPO: https://www.scylladb.com/download/?platform=ubuntu-20.04&version=scylla-5.2

.. |SCYLLA_DEB_SRC_REPO| replace:: ScyllaDB deb repo (|DEBIAN_SRC_REPO|_, |UBUNTU_SRC_REPO|_)

.. |SCYLLA_RPM_SRC_REPO| replace:: ScyllaDB rpm repo
.. _SCYLLA_RPM_SRC_REPO: https://www.scylladb.com/download/?platform=centos&version=scylla-5.2

.. |DEBIAN_NEW_REPO| replace:: Debian
.. _DEBIAN_NEW_REPO: https://www.scylladb.com/download/?platform=debian-10&version=scylla-5.4

.. |UBUNTU_NEW_REPO| replace:: Ubuntu
.. _UBUNTU_NEW_REPO: https://www.scylladb.com/download/?platform=ubuntu-20.04&version=scylla-5.4

.. |SCYLLA_DEB_NEW_REPO| replace:: ScyllaDB deb repo (|DEBIAN_NEW_REPO|_, |UBUNTU_NEW_REPO|_)

.. |SCYLLA_RPM_NEW_REPO| replace:: ScyllaDB rpm repo
.. _SCYLLA_RPM_NEW_REPO: https://www.scylladb.com/download/?platform=centos&version=scylla-5.4

.. |ROLLBACK| replace:: rollback
.. _ROLLBACK: ./#rollback-procedure

.. |SCYLLA_METRICS| replace:: Scylla Metrics Update - Scylla 5.2 to 5.4
.. _SCYLLA_METRICS: ../metric-update-5.2-to-5.4

=============================================================================
Upgrade Guide - |SCYLLA_NAME| |SRC_VERSION| to |NEW_VERSION|
=============================================================================

This document is a step-by-step procedure for upgrading from |SCYLLA_NAME| |SRC_VERSION| 
to |SCYLLA_NAME| |NEW_VERSION|, and rollback to version |SRC_VERSION| if required.

This guide covers upgrading ScyllaDB on Red Hat Enterprise Linux (RHEL), CentOS, Debian, 
and Ubuntu. See :doc:`OS Support by Platform and Version </getting-started/os-support>` 
for information about supported versions.

It also applies when using ScyllaDB official image on EC2, GCP, or Azure.

Before You Upgrade ScyllaDB
==============================

**Upgrade Your Driver**

If you're using a :doc:`ScyllaDB driver </using-scylla/drivers/cql-drivers/index>`, 
upgrade the driver before you upgrade ScyllaDB. The latest two versions of each driver 
are supported.

**Upgrade ScyllaDB Monitoring Stack**

If you're using the ScyllaDB Monitoring Stack, verify that your Monitoring Stack 
version supports the ScyllaDB version to which you want to upgrade. See 
`ScyllaDB Monitoring Stack Support Matrix <https://monitoring.docs.scylladb.com/stable/reference/matrix.html>`_.
  
We recommend upgrading the Monitoring Stack to the latest version.

**Check Feature Updates**

See the ScyllaDB Release Notes for the latest updates. The Release Notes are published 
at the `ScyllaDB Community Forum <https://forum.scylladb.com/>`_.

.. note::

   DateTieredCompactionStrategy is removed in 5.4. Migrate to TimeWindowCompactionStrategy
   **before** you upgrade from 5.2 to 5.4

.. note::
   
   In ScyllaDB 5.4, Raft-based consistent cluster management for existing 
   deployments is enabled by default.
   If you want the consistent cluster management feature to be disabled in 
   ScyllaDB 5.4, you must update the configuration **before** you upgrade 
   from 5.2 to 5.4:

   #. Set ``consistent_cluster_management: false`` in the ``scylla.yaml`` 
      configuration file on each node in the cluster.
   #. Start the upgrade procedure.

   Consistent cluster management **cannot** be disabled in version 5.4 if it was 
   enabled in version 5.2 in one of the following ways:

   * Your cluster was created in version 5.2 with the default ``consistent_cluster_management: true``
     configuration in ``scylla.yaml``.
   * You explicitly set ``consistent_cluster_management: true`` in 
     ``scylla.yaml`` in an existing cluster in version 5.2.

Upgrade Procedure
=================

A ScyllaDB upgrade is a rolling procedure that does **not** require full cluster shutdown.
For each of the nodes in the cluster, serially (i.e., one node at a time), you will:

* Check that the cluster's schema is synchronized
* Drain the node and backup the data
* Backup the configuration file
* Stop ScyllaDB
* Download and install new ScyllaDB packages
* Start ScyllaDB
* Validate that the upgrade was successful

Apply the following procedure **serially** on each node. Do not move to the next 
node before validating that the node you upgraded is up and running the new version.

**During** the rolling upgrade, it is highly recommended:

* Not to use the new |NEW_VERSION| features.
* Not to run administration functions, such as repairs, refresh, rebuild, or add 
  or remove nodes. See `sctool <https://manager.docs.scylladb.com/stable/sctool/>`_ 
  for suspending ScyllaDB Manager (only available for ScyllaDB Enterprise) scheduled 
  or running repairs.
* Not to apply schema changes.

Upgrade Steps
=============

Check the cluster schema
-------------------------

Make sure that all nodes have the schema synchronized before the upgrade. The upgrade 
procedure will fail if there is a schema disagreement between nodes.

.. code:: sh

   nodetool describecluster

Drain the nodes and backup data
-----------------------------------

Before any major procedure, like an upgrade, it is recommended to backup all the data 
to an external device. In ScyllaDB, backup is performed using the ``nodetool snapshot`` 
command. For **each** node in the cluster, run the following command:

.. code:: sh

   nodetool drain
   nodetool snapshot

Take note of the directory name that nodetool gives you, and copy all the directories 
having that name under ``/var/lib/scylla`` to an external backup device.

When the upgrade is completed on all nodes, remove the snapshot with the 
``nodetool clearsnapshot -t <snapshot>`` command to prevent running out of space.

Backup the configuration file
------------------------------

Back up the ``scylla.yaml`` configuration file and the ScyllaDB packages
in case you need to rollback the upgrade.

.. tabs::

   .. group-tab:: Debian/Ubuntu

      .. code:: sh
         
         sudo cp -a /etc/scylla/scylla.yaml /etc/scylla/scylla.yaml.backup
         sudo cp /etc/apt/sources.list.d/scylla.list ~/scylla.list-backup

   .. group-tab:: RHEL/CentOS

      .. code:: sh
         
         sudo cp -a /etc/scylla/scylla.yaml /etc/scylla/scylla.yaml.backup
         sudo cp /etc/yum.repos.d/scylla.repo ~/scylla.repo-backup


Gracefully stop the node
------------------------

.. code:: sh

   sudo service scylla-server stop

Download and install the new release
------------------------------------

Before upgrading, check what version you are running now using ``scylla --version``. 
You should take note of the current version in case you want to |ROLLBACK|_ the upgrade.

.. tabs::

   .. group-tab:: Debian/Ubuntu

        **To upgrade ScyllaDB:**

        #. Update the |SCYLLA_DEB_NEW_REPO| to |NEW_VERSION|.

        #. Install the new ScyllaDB version:

            .. code-block:: console

               sudo apt-get clean all
               sudo apt-get update
               sudo apt-get dist-upgrade scylla


        Answer ‘y’ to the first two questions.

   .. group-tab:: RHEL/CentOS

        **To upgrade ScyllaDB:**

        #. Update the |SCYLLA_RPM_NEW_REPO|_  to |NEW_VERSION|.
        #. Install the new ScyllaDB version:

            .. code:: sh

               sudo yum clean all
               sudo yum update scylla\* -y

.. note::

   If you are running a ScyllaDB official image (for EC2 AMI, GCP, or Azure), 
   you need to:
   
    * Download and install the new ScyllaDB release for Ubuntu; see 
      the Debian/Ubuntu tab above for instructions.
    * Update underlying OS packages.

   See :doc:`Upgrade ScyllaDB Image </upgrade/ami-upgrade>` for details.


Start the node
--------------

.. code:: sh

   sudo service scylla-server start

Validate
--------

#. Check cluster status with ``nodetool status`` and make sure **all** nodes, including 
   the one you just upgraded, are in ``UN`` status.
#. Use ``curl -X GET "http://localhost:10000/storage_service/scylla_release_version"`` 
   to check the ScyllaDB version. Validate that the version matches the one you upgraded to.
#. Check scylla-server log (by ``journalctl _COMM=scylla``) and ``/var/log/syslog`` to 
   validate there are no new errors in the log.
#. Check again after two minutes, to validate no new issues are introduced.

Once you are sure the node upgrade was successful, move to the next node in the cluster.

See |Scylla_METRICS|_ for more information.

Rollback Procedure
==================

.. include:: /upgrade/_common/warning_rollback.rst

The following procedure describes a rollback from |SCYLLA_NAME| |NEW_VERSION|.x to 
|SRC_VERSION|.y. Apply this procedure if an upgrade from |SRC_VERSION| to 
|NEW_VERSION| fails before completing on all nodes. 
Use this procedure only for nodes you upgraded to |NEW_VERSION|.

.. warning::

   The rollback procedure can be applied **only** if some nodes have not been 
   upgraded to |NEW_VERSION| yet.As soon as the last node in the rolling upgrade 
   procedure is started with |NEW_VERSION|, rollback becomes impossible. At that 
   point, the only way to restore a cluster to |SRC_VERSION| is by restoring it 
   from backup.

ScyllaDB rollback is a rolling procedure that does **not** require full cluster shutdown.

For each of the nodes you rollback to |SRC_VERSION|, serially (i.e., one node 
at a time), you will:

* Drain the node and stop Scylla
* Retrieve the old ScyllaDB packages
* Restore the configuration file
* Reload systemd configuration
* Restart ScyllaDB
* Validate the rollback success

Apply the following procedure **serially** on each node. Do not move to the next 
node before validating that the rollback was successful and the node is up and 
running the old version.

Rollback Steps
==============
Drain and gracefully stop the node
----------------------------------

.. code:: sh

   nodetool drain
   nodetool snapshot
   sudo service scylla-server stop

Restore and install the old release
------------------------------------

.. tabs::

   .. group-tab:: Debian/Ubuntu

        #. Restore the |SRC_VERSION| packages backed up during the upgrade.

            .. code:: sh

               sudo cp ~/scylla.list-backup /etc/apt/sources.list.d/scylla.list
               sudo chown root.root /etc/apt/sources.list.d/scylla.list
               sudo chmod 644 /etc/apt/sources.list.d/scylla.list

        #. Install:

            .. code-block::

               sudo apt-get update
               sudo apt-get remove scylla\* -y
               sudo apt-get install scylla

        Answer ‘y’ to the first two questions.

   .. group-tab:: RHEL/CentOS

        #. Restore the |SRC_VERSION| packages backed up during the upgrade procedure.

            .. code:: sh

               sudo cp ~/scylla.repo-backup /etc/yum.repos.d/scylla.repo
               sudo chown root.root /etc/yum.repos.d/scylla.repo
               sudo chmod 644 /etc/yum.repos.d/scylla.repo

        #. Install:

            .. code:: console

               sudo yum clean all
               sudo rm -rf /var/cache/yum
               sudo yum remove scylla\\*tools-core
               sudo yum downgrade scylla\\* -y
               sudo yum install scylla

Restore the configuration file
------------------------------
.. code:: sh

   sudo rm -rf /etc/scylla/scylla.yaml
   sudo cp /etc/scylla/scylla.yaml-backup /etc/scylla/scylla.yaml

Reload systemd configuration
----------------------------

You must reload the unit file if the systemd unit file is changed.

.. code:: sh

   sudo systemctl daemon-reload

Start the node
--------------

.. code:: sh

   sudo service scylla-server start

Validate
--------
Check the upgrade instructions above for validation. Once you are sure the node 
rollback is successful, move to the next node in the cluster.
