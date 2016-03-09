===========================
Open vSwitchS Topology Node
===========================

Open vSwitch Kernel Module
--------------------------

Open vSwitch requires the openvswitch kernel module to be loaded in the host
machine.

The OVS images should work in user space mode without the module but this
experimental mode was not working in Docker at the time of writing.

The recommended way to run OVS is to first install the module included with the
corresponding OVS release. Check the docker image's OVS version by looking at
its `tag <https://hub.docker.com/r/topology/openvswitch/tags/>`_ or spawning a
container. Then `download <http://openvswitch.org/releases/>`_ the corresponding
OVS version, build OVS and load the kernel module:

::

   ./configure --with-linux=/lib/modules/`uname -r`/build
   make
   sudo make modules_install
   sudo rmmod openvswitch  # If already loaded
   sudo modprobe openvswitch

Check the `OVS FAQ <https://github.com/openvswitch/ovs/blob/master/FAQ.md#q-what-linux-kernel-versions-does-each-open-vswitch-release-work-with>`_ for information on kernel version support.


Using Open vSwitch Nodes
------------------------

The OpenvSwitch node in Topology looks for the
``socketplane/openvswitch:latest`` docker image by default. The only shell
available to OpenvSwitch is the busybox shell ``sh``, which allows running:

- ``ovs-appclt`` commands, to control the OVS daemon.
- ``ovs-oftcl``, to configure OpenFlow.
- ``ovs-vsctl`` to configure the switch.

For example, you may bring up a bridge with:

::

   # Create a bridge
   sw1('ovs-vsctl add-br br0')

   # Bring up ovs interface
   sw1('ip link set br0 up')

   # Add the front ports
   sw1('ovs-vsctl add-port br0 1')
   sw1('ovs-vsctl add-port br0 2')

Interfaces should be up before the call to add-port. You can create them and
bring them up manually or using the topology definition:

::

   [up=True] sw1:1
   [up=True] sw1:2


Debugging OpenvSwitch Node
--------------------------

Open vSwitch is started by supervisor.

- If you have access to the running container, supervisorctl allows you to
  check the status and logs of the ovs-switchd and ovsdb-server processes.
- If the Topology startup fails, stdout and stderr logs for every supervisor
  process are kept in the container at the /tmp folder, which is shared with
  your host machine at ``/tmp/topology_<NODE_NAME>_<NODE_ID>``, so that you are
  able to check those logs afterwards.
- Check the ``supervisord.conf`` file for details on how the services are being
  started.


The docker image
----------------

The following section explains the process used to build the docker OVS image.
It may be useful for advanced users when creating or customizing the docker
image but not when writing tests using the default features.

The OVS docker switch was built making use of
`socketplane's docker-ovs images <https://github.com/socketplane/docker-ovs>`_.

Each folder corresponds to an OpenVswitch version and includes the Dockerfile
and two required files.

- OVS is brought up by supervisor. The ``supervisord.conf`` file is copied to
  the container to be run by supervisor.
- ``configure-ovs.sh`` executes some OVS startup commands.

Depending on you environment, you may need to set a proxy in the building
container, by setting the http_proxy and https_proxy variables in the
Dockerfile:

::

   ENV http_proxy http://proxy.houston.hp.com:8080/
   ENV https_proxy http://proxy.houston.hp.com:8080/

Then simply build the Docker image with:

::

   cd version_folder
   docker build -t openvswitch:latest .

This creates an OVS docker image with the required capabilities. The image auto
starts supervisord with ``nodaemon=true``. This is undesirable in topology since
it blocks sdtin, and should be disabled in the ``supervisord.conf`` file.
