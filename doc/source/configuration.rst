.. _configuration:

Configuration
=============

Nodepool reads its configuration from ``/etc/nodepool/nodepool.yaml``
by default.  The configuration file follows the standard YAML syntax
with a number of sections defined with top level keys.  For example, a
full configuration file may have the ``labels``, ``providers``, and
``targets`` sections. If building images using diskimage-builder, the
``diskimages`` section is also required::

  labels:
    ...
  diskimages:
    ...
  providers:
    ...
  targets:
    ...

The following sections are available.  All are required unless
otherwise indicated.

script-dir
----------
When creating an image to use when launching new nodes, Nodepool will
run a script that is expected to prepare the machine before the
snapshot image is created.  The ``script-dir`` parameter indicates a
directory that holds all of the scripts needed to accomplish this.
Nodepool will copy the entire directory to the machine before invoking
the appropriate script for the image being created.

Example::

  script-dir: /path/to/script/dir

.. _elements-dir:

elements-dir
------------

If an image is configured to use diskimage-builder and glance to locally
create and upload images, then a collection of diskimage-builder elements
must be present. The ``elements-dir`` parameter indicates a directory
that holds one or more elements.

Example::

  elements-dir: /path/to/elements/dir

images-dir
----------

When we generate images using diskimage-builder they need to be
written to somewhere. The ``images-dir`` parameter is the place to
write them.

Example::

  images-dir: /path/to/images/dir

dburi
-----
Indicates the URI for the database connection.  See the `SQLAlchemy
documentation
<http://docs.sqlalchemy.org/en/latest/core/engines.html#database-urls>`_
for the syntax.  Example::

  dburi: 'mysql+pymysql://nodepool@localhost/nodepool'

nodepoold
---------
This section is optional.

Parameters to alter the global Nodepool behavior.

  ``delete-delay`` (int)
  Time to wait before deleting a node that has completed its job.
  In seconds. Default 60.

cron
----
This section is optional.

Nodepool runs several periodic tasks.  The ``image-update`` task
creates a new image for each of the defined images, typically used to
keep the data cached on the images up to date.  The ``cleanup`` task
deletes old images and servers which may have encountered errors
during their initial deletion.  The ``check`` task attempts to log
into each node that is waiting to be used to make sure that it is
still operational.  The following illustrates how to change the
schedule for these tasks and also indicates their default values::

  cron:
    image-update: '14 2 * * *'
    cleanup: '27 */6 * * *'
    check: '*/15 * * * *'

zmq-publishers
--------------
Lists the ZeroMQ endpoints for the Jenkins masters.  Nodepool uses
this to receive real-time notification that jobs are running on nodes
or are complete and nodes may be deleted.  Example::

  zmq-publishers:
    - tcp://jenkins1.example.com:8888
    - tcp://jenkins2.example.com:8888

gearman-servers
---------------
Lists the Zuul Gearman servers that should be consulted for real-time
demand.  Nodepool will use information from these servers to determine
if additional nodes should be created to satisfy current demand.
Example::

  gearman-servers:
    - host: zuul.example.com
      port: 4730

The ``port`` key is optional (default: 4730).

.. _labels:

labels
------

Defines the types of nodes that should be created.  Maps node types to
the images that are used to back them and the providers that are used
to supply them.  Jobs should be written to run on nodes of a certain
label (so targets such as Jenkins don't need to know about what
providers or images are used to create them).  Example::

  labels:
    - name: my-precise
      image: precise
      min-ready: 2
      providers:
        - name: provider1
        - name: provider2
    - name: multi-precise
      image: precise
      subnodes: 2
      min-ready: 2
      ready-script: setup_multinode.sh
      providers:
        - name: provider1

**required**

  ``name``
    Unique name used to tie jobs to those instances.

  ``image``
    Refers to providers images, see :ref:`images`.

  ``providers`` (list)
    Required if any nodes should actually be created (e.g., the label is not
    currently disabled, see ``min-ready`` below).

**optional**

  ``min-ready`` (default: 2)
    Minimum instances that should be in a ready state. Set to -1 to have the
    label considered disabled. ``min-ready`` is best-effort based on available
    capacity and is not a guaranteed allocation.

  ``subnodes``
    Used to configure multi-node support.  If a `subnodes` key is supplied to
    an image, it indicates that the specified number of additional nodes of the
    same image type should be created and associated with each node for that
    image.

    Only one node from each such group will be added to the target, the
    subnodes are expected to communicate directly with each other.  In the
    example above, for each Precise node added to the target system, two
    additional nodes will be created and associated with it.

  ``ready-script``
    A script to be used to perform any last minute changes to a node after it
    has been launched but before it is put in the READY state to receive jobs.
    For more information, see :ref:`scripts`.

.. _diskimages:

diskimages
----------

Lists the images that are going to be built using diskimage-builder.
Image keyword defined on labels section will be mapped to the
images listed on diskimages. If an entry matching the image is found
this will be built using diskimage-builder and the settings found
on this configuration. If no matching image is found, image
will be built using the provider snapshot approach::

  diskimages:
  - name: devstack-precise
    elements:
      - ubuntu
      - vm
      - puppet
      - node-devstack
    release: precise
    env-vars:
        DIB_DISTRIBUTION_MIRROR: http://archive.ubuntu.com
        DIB_IMAGE_CACHE: /opt/dib_cache


**required**

  ``name``
    Identifier to reference the disk image in :ref:`images` and :ref:`labels`.

**optional**

  ``release``
    Specifies the distro to be used as a base image to build the image using
    diskimage-builder.

  ``elements`` (list)
    Enumerates all the elements that will be included when building the image,
    and will point to the :ref:`elements-dir` path referenced in the same
    config file.

  ``env-vars`` (dict)
    Arbitrary environment variables that will be available in the spawned
    diskimage-builder child process.

.. _provider:

provider
---------

Lists the OpenStack cloud providers Nodepool should use.  Within each
provider, the Nodepool image types are also defined (see
:ref:`images` for details).  Example::

  providers:
    - name: provider1
      username: 'username'
      password: 'password'
      auth-url: 'http://auth.provider1.example.com/'
      project-id: 'project'
      service-type: 'compute'
      service-name: 'compute'
      region-name: 'region1'
      max-servers: 96
      rate: 1.0
      availability-zones:
        - az1
      boot-timeout: 120
      launch-timeout: 900
      template-hostname: '{image.name}-{timestamp}.template.openstack.org'
      pool: 'public'
      image-type: qcow2
      networks:
        - net-id: 'some-uuid'
        - net-label: 'some-network-name'
      images:
        - name: trusty
          base-image: 'Trusty'
          min-ram: 8192
          name-filter: 'something to match'
          setup: prepare_node.sh
          reset: reset_node.sh
          username: jenkins
          user-home: '/home/jenkins'
          private-key: /var/lib/jenkins/.ssh/id_rsa
          meta:
              key: value
              key2: value
        - name: precise
          base-image: 'Precise'
          min-ram: 8192
          setup: prepare_node.sh
          reset: reset_node.sh
          username: jenkins
          user-home: '/home/jenkins'
          private-key: /var/lib/jenkins/.ssh/id_rsa
        - name: devstack-trusty
          min-ram: 30720
          diskimage: devstack-trusty
          username: jenkins
          private-key: /home/nodepool/.ssh/id_rsa
    - name: provider2
      username: 'username'
      password: 'password'
      auth-url: 'http://auth.provider2.example.com/'
      project-id: 'project'
      service-type: 'compute'
      service-name: 'compute'
      region-name: 'region1'
      max-servers: 96
      rate: 1.0
      template-hostname: '{image.name}-{timestamp}-nodepool-template'
      images:
        - name: precise
          base-image: 'Fake Precise'
          min-ram: 8192
          setup: prepare_node.sh
          reset: reset_node.sh
          username: jenkins
          user-home: '/home/jenkins'
          private-key: /var/lib/jenkins/.ssh/id_rsa
          meta:
              key: value
              key2: value

**required**

  ``name``

  ``username``

  ``password``

  ``project-id``
    Some clouds may refer to the ``project-id`` as ``tenant-id``.

  ``auth-url``
    Keystone URL.

  ``max-servers``
    Maximum number of servers spawnable on this provider.

**optional**

  ``availability-zones`` (list)
    Without it nodepool will rely on nova to schedule an availability zone.

    If it is provided the value should be a list of availability zone names.
    Nodepool will select one at random and provide that to nova. This should
    give a good distribution of availability zones being used. If you need more
    control of the distribution you can use multiple logical providers each
    providing a different list of availabiltiy zones.

  ``boot-timeout``
    In seconds. Default 60.

  ``launch-timeout``
    In seconds. Default 3600.

  ``image-type``
    Specifies the image type supported by this provider.  The disk images built
    by diskimage-builder will output an image for each ``image-type`` specified
    by a provider using that particular diskimage.

    The default value is ``qcow2``, and values of ``vhd``, ``raw`` are also
    expected to be valid if you have a sufficiently new diskimage-builder.

  ``keypair``
    Default None

  ``networks`` (dict)
    Specify custom Neutron networks that get attached to each node. You can
    specify Neutron networks using either the ``net-id`` or ``net-label``. If
    only the ``net-label`` is specified the network UUID is automatically
    queried via the Nova os-tenant-networks API extension (this requires that
    the cloud provider has deployed this extension).

  ``pool``
    Specify a floating ip pool in cases where the 'public' pool is unavailable
    or undesirable.

  ``api_timeout``
    Timeout for the Nova client in seconds.

  ``service-type``

  ``service-name``

  ``region-name``

  ``template-hostname``
    Hostname template to use for the spawned instance.
    Default ``{image.name}-{timestamp}.template.openstack.org``

  ``rate``
    In seconds. Default 1.0.

.. _images:

images
~~~~~~

Example::

  images:
    - name: precise
      base-image: 'Precise'
      min-ram: 8192
      name-filter: 'something to match'
      setup: prepare_node.sh
      reset: reset_node.sh
      username: jenkins
      private-key: /var/lib/jenkins/.ssh/id_rsa
      meta:
          key: value
          key2: value

**required**

  ``name``
    Identifier to refer this image from :ref:`labels` and :ref:`provider`
    sections.

    If the resulting images from different providers ``base-image`` should be
    equivalent, give them the same name; e.g. if one provider has a ``Fedora
    20`` image and another has an equivalent ``Fedora 20 (Heisenbug)`` image,
    they should use a common ``name``.  Otherwise select a unique ``name``.

  ``base-image``
    UUID or string-name of the image to boot as specified by the provider.

  ``min-ram``
    Determine the flavor of ``base-image`` to use (e.g. ``m1.medium``,
    ``m1.large``, etc).  The smallest flavor that meets the ``min-ram``
    requirements will be chosen. To further filter by flavor name, see optional
    ``name-filter`` below.

**optional**

  ``name-filter``
    Additional filter complementing ``min-ram``, will be required to match on
    the flavor-name (e.g. Rackspace offer a "Performance" flavour; setting
    `name-filter` to ``Performance`` will ensure the chosen flavor also
    contains this string as well as meeting `min-ram` requirements).

  ``setup``
     Script to run to prepare the instance.

     Used only when not building images using diskimage-builder, in that case
     settings defined in the ``diskimages`` section will be used instead. See
     :ref:`scripts` for setup script details.

  ``reset``
     See :ref:`scripts`.

  ``diskimages``
     See :ref:`diskimages`.

  ``username``
    Nodepool expects that user to exist after running the script indicated by
    ``setup``. Default ``jenkins``

  ``private-key``
    Default ``/var/lib/jenkins/.ssh/id_rsa``

  ``config-drive`` (boolean)
    Whether config drive should be used for the image.

  ``meta`` (dict)
    Arbitrary key/value metadata to store for this server using the Nova
    metadata service. A maximum of five entries is allowed, and both keys and
    values must be 255 characters or less.

.. _targets:

targets
-------

Lists the Jenkins masters to which Nodepool should attach nodes after
they are created.  Nodes of each label will be evenly distributed
across all of the targets which are on-line::

  targets:
    - name: jenkins1
      jenkins:
        url: https://jenkins1.example.org/
        user: username
        apikey: key
        credentials-id: id
      hostname: '{label.name}-{provider.name}-{node_id}.slave.openstack.org'
      subnode-hostname: '{label.name}-{provider.name}-{node_id}-{subnode_id}.slave.openstack.org'
    - name: jenkins2
      jenkins:
        url: https://jenkins2.example.org/
        user: username
        apikey: key
        credentials-id: id
      hostname: '{label.name}-{provider.name}-{node_id}'
      subnode-hostname: '{label.name}-{provider.name}-{node_id}-{subnode_id}'

**required**

  ``name``
  Identifier for the system an instance is attached to.

**optional**

  ``hostname``
    Default ``{label.name}-{provider.name}-{node_id}.slave.openstack.org``

  ``subnode-hostname``
    Default ``{label.name}-{provider.name}-{node_id}-{subnode_id}.slave.openstack.org``

  ``rate``
    In seconds. Default 1.0

  ``jenkins`` (dict)
    ``url``
      Url to the Jenkins REST API.

    ``user``
      Jenkins username.

    ``apikey``
      API key generated by Jenkins (not the user password).

    ``credentials-id`` (optional)
      If provided, Nodepool will configure the Jenkins slave to use the Jenkins
      credential identified by that ID, otherwise it will use the username and
      ssh keys configured in the image.

    ``test-job`` (optional)
      Setting this would cause a newly created instance to be in a TEST state.
      The job name given will then be executed with the node name as a
      parameter.

      If the job succeeds, move the node into READY state and relabel it with
      the appropriate label (from the image name).

      If it fails, immediately delete the node.

      If the job never runs, the node will eventually be cleaned up by the
      periodic cleanup task.
