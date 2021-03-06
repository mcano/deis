:title: Upgrading Deis
:description: Guide to upgrading Deis to a new release.


.. _upgrading-deis:

Upgrading Deis
==============

There are currently two strategies for upgrading a Deis cluster:

* In-place Upgrade (recommended)
* Migration Upgrade

Before attempting an upgrade, it is strongly recommended to :ref:`backup your data <backing_up_data>`.

In-place Upgrade
----------------

An in-place upgrade swaps out platform containers for newer versions on the same set of hosts,
leaving your applications and platform data intact.  This is the easiest and least disruptive upgrade strategy.
The general approach is to use ``deisctl`` to uninstall all platform components, update the platform version
and then reinstall platform components.

.. important::

    Always use a version of ``deisctl`` that matches the Deis release.
    Verify this with ``deisctl --version``.

Use the following steps to perform an in-place upgrade of your Deis cluster.

First, use the current ``deisctl`` to stop and uninstall the Deis platform.

.. code-block:: console

    $ deisctl --version  # should match the installed platform
    1.0.2
    $ deisctl stop platform && deisctl uninstall platform

There are important security fixes since Deis 1.0.2 that require upgrading
to CoreOS 494.1.0 or later, and configuring Docker to access deis-registry. See
:ref:`upgrading-coreos` first, then open a shell to each node:

.. code-block:: console

    $ ssh deis-1.example.com  # repeat these steps for each node
    $ sudo -i
    $ mkdir -p /etc/systemd/system/docker.service.d
    $ cat <<EOF > /etc/systemd/system/docker.service.d/50-insecure-registry.conf
    [Service]
    Environment="DOCKER_OPTS=--insecure-registry 10.0.0.0/8 --insecure-registry 172.16.0.0/12 --insecure-registry 192.168.0.0/16"
    EOF
    $ reboot  # one node at a time, to avoid etcd failures

Finally, update ``deisctl`` to the new version and reinstall:

.. code-block:: console

    $ curl -sSL http://deis.io/deisctl/install.sh | sh -s 1.2.0
    $ deisctl --version  # should match the desired platform
    1.2.0
    $ deisctl config platform set version=v1.2.0
    $ deisctl install platform
    $ deisctl start platform

.. attention::

    In-place upgrades incur approximately 10-30 minutes of downtime for deployed applications, the router mesh
    and the platform control plane.  Please plan your maintenance windows accordingly.


Migration Upgrade
-----------------

This upgrade method provisions a new cluster running in parallel to the old one. Applications are
migrated to this new cluster one-by-one, and DNS records are updated to cut over traffic on a
per-application basis. This results in a no-downtime controlled upgrade, but has the caveat that no
data from the old cluster (users, releases, etc.) is retained. Future ``deisctl`` tooling will have
facilities to export and import this platform data.

.. note::

    Migration upgrades are useful for moving Deis to a new set of hosts,
    but should otherwise be avoided due to the amount of manual work involved.

.. important::

    In order to migrate applications, your new cluster must have network access
    to the registry component on the old cluster

Enumerate Existing Applications
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
Each application will need to be deployed to the new cluster manually.
Log in to the existing cluster as an admin user and use the ``deis`` client to
gather information about your deployed applications.

List all applications with:

.. code-block:: console

    $ deis apps:list

Gather each application's version with:

.. code-block:: console

    $ deis apps:info -a <app-name>

Provision servers
^^^^^^^^^^^^^^^^^
Follow the Deis documentation to provision a new cluster using your desired target release.
Be sure to use a new etcd discovery URL so that the new cluster doesn't interfere with the running one.

Upgrade Deis clients
^^^^^^^^^^^^^^^^^^^^
If changing versions, make sure you upgrade your ``deis`` and ``deisctl`` clients
to match the cluster's release.

Register and login to the new controller
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
Register an account on the new controller and login.

.. code-block:: console

    $ deis register http://deis.newcluster.example.org
    $ deis login http://deis.newcluster.example.org

Migrate applications
^^^^^^^^^^^^^^^^^^^^
The ``deis pull`` command makes it easy to migrate existing applications from
one cluster to another.  However, you must have network access to the existing
cluster's registry component.

Migrate a single application with:

.. code-block:: console

    $ deis create <app-name>
    $ deis pull registry.oldcluster.example.org:5000/<app-name>:<version>

This will move the application's Docker image across clusters, ensuring the application
is migrated bit-for-bit with an identical build and configuration.

Now each application is running on the new cluster, but they are still running (and serving traffic)
on the old cluster.  Use ``deis domains:add`` to tell Deis that this application can be accessed
by its old name:

.. code-block:: console

    $ deis domains:add oldappname.oldcluster.example.org

Repeat for each application.

Test applications
^^^^^^^^^^^^^^^^^
Test to make sure applications work as expected on the new Deis cluster.

Update DNS records
^^^^^^^^^^^^^^^^^^
For each application, create CNAME records to point the old application names to the new. Note that
once these records propagate, the new cluster is serving live traffic. You can perform cutover on a
per-application basis and slowly retire the old cluster.

If an application is named 'happy-bandit' on the old Deis cluster and 'jumping-cuddlefish' on the
new cluster, you would create a DNS record that looks like the following:

.. code-block:: console

    happy-bandit.oldcluster.example.org.        CNAME       jumping-cuddlefish.newcluster.example.org

Retire the old cluster
^^^^^^^^^^^^^^^^^^^^^^
Once all applications have been validated, the old cluster can be retired.


.. _upgrading-coreos:

Upgrading CoreOS
----------------

By default, Deis disables CoreOS automatic updates. This is partially because of problems we've seen
with etcd/fleet version incompatibilities as hosts in the cluster are upgraded one-by-one.
Additionally, because Deis customizes the CoreOS cloud-config file, upgrading the CoreOS host to
a new version without accounting for changes in the cloud-config file could cause Deis to stop
functioning properly.

.. important::

  Enabling updates for CoreOS will result in the machine upgrading to the latest CoreOS release
  available in a particular channel. Sometimes, new CoreOS releases make changes that will break
  Deis. It is always recommended to provision a Deis release with the CoreOS version specified
  in that release's provision scripts or documentation.

While typically not recommended, it is possible to trigger an update of a CoreOS machine. Some
Deis releases may recommend a CoreOS upgrade - in these cases, the release notes for a Deis release
will point to this documentation.

To update CoreOS, run the following commands:

.. code-block:: console

    $ ssh core@<server ip>
    $ sudo su
    $ echo GROUP=stable > /etc/coreos/update.conf
    $ systemctl unmask update-engine.service
    $ systemctl start update-engine.service
    $ update_engine_client -update
    $ systemctl stop update-engine.service
    $ systemctl mask update-engine.service
    $ reboot

.. warning::

  You should only upgrade one host at a time. Removing multiple hosts from the cluster
  simultaneously can result in failure of the etcd cluster. Ensure the recently-rebooted host
  has returned to the cluster with ``fleetctl list-machines`` before moving on to the next host.

You can check the CoreOS version by running the following command on the CoreOS machine:

.. code-block:: console

    $ cat /etc/os-release

Or from your local machine:

.. code-block:: console

    $ ssh core@<server ip> 'cat /etc/os-release'
