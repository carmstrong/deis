:title: Remove Ceph dependency
:description: Configuring the cluster to remove Ceph from the control plane.

.. _remove-ceph-dependency:

Remove Ceph dependency
======================

.. include:: ../_includes/_ceph-dependency-description.rst

This guide is intended to assist users who are interested in removing the Ceph
dependency of the Deis control plane.

.. note::

  This guide was adapted from content graciously provided by Deis community member
  `Arne-Christian Blystad`_.

Requirements
------------

External services are required to replace the internal store components:

* S3-compatible blob store (like `Amazon S3`_)
* PostgreSQL database (like `Amazon RDS`_)
* Log drain service with syslog log format compatibility (like `Papertrail`_)

Understanding component changes
-------------------------------

Either directly or indirectly, all components in the :ref:`control-plane`
require Ceph (:ref:`store`). Some components require changes to accommodate
the removal of Ceph. The necessary changes are described below.

Logger
^^^^^^

The :ref:`logger` component provides a syslog-compatible endpoint to consume
application logs, which it writes to a shared Ceph filesystem. These logs are
read by the :ref:`controller` component. The :ref:`logspout` talks to the Docker
daemon on each host, listens for log events from running applications, and ships
them to the logger.

The Logger component is not necessary in a Ceph-less Deis cluster. Instead of
using the Logger, we will route all the logs directly to another syslog
compatible endpoint.

Database
^^^^^^^^

The :ref:`database` runs PostgreSQL and uses the Ceph S3 API (provided by
``deis-store-gateway``) to store PostgreSQL backups and WAL logs.
Should the host running database fail, the database component will fail over to
a new host, start up, and replay backups and WAL logs to recover to its
previous state.

We will not be using the database component in the Ceph-less cluster, and will
instead rely on an external database.

Controller
^^^^^^^^^^

The :ref:`controller` component hosts the API that the Deis CLI consumes. The controller
mounts the same Ceph filesystem that the logger writes to. When users run ``deis logs``
to view an application's log files, the controller reads from this shared filesystem.

A Ceph-less cluster will not store logs (instead sending them to an external service),
so the ``deis logs`` command will not work for users.

Registry
^^^^^^^^

The :ref:`registry` component is an instance of the offical Docker registry, and
is used to store application releases. The registry supports any S3 store, so
a Ceph-less cluster will simply reconfigure registry to use another store (typically
Amazon S3 itself).

Builder
^^^^^^^

The :ref:`builder` component is responsible for building applications deployed
to Deis via the ``git push`` workflow. It pushes to registry to store releases,
so it will require no changes.

Store
^^^^^

The :ref:`store` components implement Ceph itself. In a Ceph-less cluster, we
will skip the installation and starting of these components. We will also
mock a fake ``deis-store-volume`` component so that the controller and logger
are happy.

Deploying the cluster
---------------------

This guide assumes a typical deployment on AWS by following the :ref:`deis_on_aws`
guide.

Deploy an AWS cluster
^^^^^^^^^^^^^^^^^^^^^

Follow the :ref:`deis_on_aws` installation documentation through the "Configure
DNS" portion.

Configure log shipping
^^^^^^^^^^^^^^^^^^^^^^

The :ref:`logspout` component must be configured to ship logs to somewhere other
than the :ref:`logger` component.

.. code-block:: console

    $ HOST=logs.somewhere.com
    $ PORT=98765
    $ deisctl config logs set host=${HOST} port=${PORT}

Configure registry
^^^^^^^^^^^^^^^^^^

The :ref:`registry` component won't start until it's configured with an S3 store.

.. code-block:: console

    $ BUCKET=MYS3BUCKET
    $ AWS_ACCESS_KEY=something
    $ AWS_SECRET_KEY=something
    $ deisctl config registry set s3bucket=${BUCKET} \
                                  bucketName=${BUCKET} \
                                  s3accessKey=${AWS_ACCESS_KEY} \
                                  s3secretKey=${AWS_SECRET_KEY} \
                                  s3region=eu-west-1 \
                                  s3path=/ \
                                  s3encrypt=false \
                                  s3secure=false
    $ deisctl config store set gateway/accessKey=${AWS_ACCESS_KEY} \
                               gateway/secretKey=${AWS_SECRET_KEY} \
                               gateway/host=s3.amazonaws.com \
                               gateway/port=80

Configure database settings
^^^^^^^^^^^^^^^^^^^^^^^^^^^

Since we won't be running the :ref:`database`, we need to configure these settings
so the controller knows where to connect.

.. code-block:: console

    $ HOST=something.rds.amazonaws.com
    $ DB_USER=deis
    $ DB_PASS=somethingsomething
    $ DATABASE=deis
    $ deisctl config database set engine=postgresql_psycopg2 \
                                  host=${HOST} \
                                  port=5432 \
                                  name=${DATABASE } \
                                  user=${DB_USER} \
                                  password=${DB_PASS}

Fake deis-store-volume
^^^^^^^^^^^^^^^^^^^^^^

The ``deis-store-volume`` component mounts the shared Ceph filesystem used by
both the :ref:`logger` and :ref:`controller`. For the controller to run, it needs
to think that the log directory exists, so we create a fake ``deis-store-volume``
component.

This code snippet is intended to be run on a CoreOS host because it uses
``fleetctl`` directly.

.. code-block:: console

    $ cat << EOF > /tmp/deis-store-volume.service
    [Unit]
    Description=dummy deis-store-volume
    [Service]
    EnvironmentFile=/etc/environment
    ExecStart=/usr/bin/mkdir -p /var/lib/deis/store/logs
    RemainAfterExit=yes
    Type=oneshot
    [Install]
    WantedBy=multi-user.target
    [X-Fleet]
    Global=true
    EOF
    $ fleetctl load /tmp/deis-store-volume.service
    $ fleetctl start deis-store-volume.service
    $ rm /tmp/deis-store-volume.service

Deploy the platform
^^^^^^^^^^^^^^^^^^^

Since we won't be deploying many of the typical Deis components, we cannot
use ``deisctl install platform`` or ``deisctl start platform`` -- instead, we
have to specify the individual components that we'd like to install and start.

Follow :ref:`install_deis_platform` **until** the ``deisctl install platform``
step. Then, run the following script to install a subset of the components:

  .. code-block:: console

    SERVICES=("logspout" "registry@1" "controller" "builder" "publisher" "router@1" "router@2" "router@3")
    for SERVICE in ${SERVICES[@]}; do
        echo "install ${SERVICE}"
        deisctl install ${SERVICE}
    done
    echo "Starting Deis...This takes some time..."
    for SERVICE in ${SERVICES[@]}; do
        echo "start ${SERVICE}"
        deisctl start ${SERVICE}
    done

Confirm installation
^^^^^^^^^^^^^^^^^^^^

That's it! Deis is now running without Ceph. Issue a ``deisctl list`` to confirm
that the services are started, and see :ref:`using_deis` to start using the cluster.

Upgrading Deis
--------------

As expected, upgrading Deis is not quite as easy as the :ref:`upgrading-deis`
documentation states for a standard cluster -- instead of referencing ``platform``,
it's necessary to target only the components we targeted in the "Deploy the platform"
step above.

The general procedure, however, is the same as stated in the upgrade documentation.

.. _`Amazon RDS`: http://aws.amazon.com/rds/
.. _`Amazon S3`: http://aws.amazon.com/s3/
.. _`Arne-Christian Blystad`: https://github.com/blystad
.. _`Papertrail`: https://papertrailapp.com/
