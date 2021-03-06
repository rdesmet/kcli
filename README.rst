kcli repository
===============

|Build Status| |Pypi|

This script is meant to interact with a local/remote libvirt daemon and
to easily deploy from templates ( optionally using cloudinit). It will
also report ips for any vm connected to a dhcp enabled libvirt network
and generally for every vm deployed from this client.

It started because i switched from ovirt and needed a tool similar to
`ovirt.py <https://github.com/karmab/ovirt>`__

Why should i use this instead of vagrant
----------------------------------------

-  easy syntax to launch single or multiple vms
-  cloudinit based customization, not over ssh
-  no need of using custom images, the public ones will do
-  spice/vnc consoles and tcp serial ones

demo
----

|asciicast|

installation
------------

install requirements. you will also need to grab *mkisofs* for cloudinit
isos to get generated Console access is based on remote-viewer For
instance if using a rhel based distribution:

::

    yum -y install gcc libvirt-devel python-devel genisoimage qemu-kvm telnet python-pip

If using a debian based distribution:

::

    apt-get -y install python-pip pkg-config libvirt-dev genisoimage qemu-kvm telnet libvirt-bin

then you can install from pypi

::

    pip install kcli

To deploy from templates, grab images at
`openstack <http://docs.openstack.org/image-guide/obtain-images.html>`__
## configuration

If you want to only use your local libvirt daemon, no configuration is
needed. If you want to generate a basic settings file, you can use the
following command:

::

    kcli bootstrap -f

You can also go through wizard

::

    kcli bootstrap

And for advanced bootstrapping, you can specify a target name, host, a
pool with a path

::

    kcli bootstrap -a -n twix -H 192.168.0.6 --pool vms --poolpath /home/vms

Or even use an existing disk for LVM based images

::

    kcli bootstrap -a -n twix -H 192.168.0.6 --pool vms --poolpath /dev/vdb --pooltype lvm

Otherwise you will have to declare your settings in ~/kcli.yml. For
instance,

::

    default:
     client: twix
     numcpus: 2
     diskthin1: true
     memory: 512
     disks:
      - size: 10
     protocol: ssh
     cloudinit: true
     nets: 
      - private1

    twix:
     host: 192.168.0.6
     pool: images

    bumblefoot:
     host: 192.168.0.4
     pool: images

replace with your own client in default section and indicate host and
protocol in the corresponding client section. Note that most of the
parameters are actually optional, and can be overriden in the profile
section ( or in a plan file)

profile configuration
---------------------

You can use the file ~/kvirt\_profiles.yml to specify profiles (number
of cpus, memory, size of disk,network,....) to use when deploying a vm.

The `samples
directory <https://github.com/karmab/kcli/tree/master/samples>`__
contains examples to get you started

How to use
----------

-  get info on your kvm setup
-  ``kcli report``
-  list vms, along with their private ip ( and plan if applicable)
-  ``kcli list``
-  list templates
-  ``kcli list -t``
-  create vm from profile base7
-  ``kcli create -p base7 myvm``
-  delete vm
-  ``kcli delete vm1``
-  get detailed info on a specific vm
-  ``kcli info vm1``
-  start vm
-  ``kcli start vm1``
-  stop vm
-  ``kcli start vm1``
-  get remote-viewer console
-  ``kcli console vm1``
-  get serial console ( over tcp!!!). Note that it will only work with
   vms created with kcli and will require telnet client to be installed
   on host
-  ``kcli console -s vm1``
-  deploy multiple vms using plan x defined in x.yml file
-  ``kcli plan -f x.yml x``
-  delete all vms from plan x
-  ``kcli plan -d x``
-  add 5GB disk to vm1
-  ``kcli add -s 5 vm1``
-  update to 2GB memory vm1
-  ``kcli update -m 2048 vm1``
-  update internal ip ( usefull for ansible inventory over existing
   bridged vms)
-  ``kcli update -1 192.168.0.40 vm1``
-  clone vm1 to new vm2
-  ``kcli clone -b vm1 vm2``
-  connect by ssh to the vm ( retrieving ip and adjusting user based on
   the template)
-  ``kcli ssh vm1``
-  switch active client to bumblefoot
-  ``kcli switch bumblefoot``
-  add a new network
-  ``kcli network -c 192.168.7.0/24 --dhcp mynet``

cloudinit stuff
---------------

if cloudinit is enabled (it is by default), a custom iso is generated on
the fly for your vm ( using mkisofs) and uploaded to your kvm instance (
using the API). the iso handles static networking configuration,
hostname setting, inyecting ssh keys and running specific commands

Also note that if you use cloudinit but dont specify ssh keys to inject,
the default ~/.ssh/id\_rsa.pub will be used, if present.

Using plans
-----------

you can also define plan files in yaml with a list of vms to deploy (
look at the sample) and deploy it with kcli plan

You can point at an existing profile within your plans, define all
parameters for the vms, or combine both approaches.

Specific scripts and ips arrays can be used directly in the plan file (
or in profiles one)

The samples directory contains examples to get you started

Note that the description of the vm will automatically be set to the
plan name, and this value will be used when deleting the entire plan as
a way to locate matching vms.

When launching a plan, the plan name is optional. If not is provided,
the kvirt keyword will be used.

If a file with the plan isnt specified with -f , the file kcli\_plan.yml
in the current directory will be used, if available.

For an advanced use of plans along with scripts, you can check the
`uci <uci/README.md>`__ page to deploy all upstream projects associated
with Red Hat Cloud Infrastructure products ( or downstream versions too)

available parameters
--------------------

those parameters can be set either in your config, profile or plan files

-  *numcpus* Defaults to 2
-  *memory* Defaults to 512
-  *guestid* Defaults to guestrhel764
-  *pool* Defaults to default
-  *template* Should point to your base cloud image(optional)
-  *disks* Array of disks to define. For each of them, you can specify
   size, thin ( as boolean) and interface ( either ide or virtio).If you
   omit parameters, default values will be used from config or profile
   file ( You can actually let the entire entry blank or just indicate a
   size number directly). For instance:

::

    disks:
     - size: 20
     - size: 10
       thin: False
       format: ide

Within a disk section, you can use the word size, thin and format as
keys

-  *diskthin* Value used when not specified in the disk entry. Defaults
   to true
-  *diskinterface* Value used when not specified in the disk entry.
   Defaults to virtio. Could also be ide, if vm lacks virtio drivers
-  *nets* Array of networks. Defaults to ['default']. You can mix simple
   strings pointing to the name of your network and more complex
   information provided as hash. For instance:

::

    nets:
     - private1
     - name: private2:
       nic: eth1
       mask: 255.255.255.0
       gateway: 192.168.0.1

Within a net section, you can use name, nic, ip, mask and gateway as
keys.

Note that up to 8 ips can also be provided on command line when creating
a single vm ( with the flag -1, -2, -3,-4,...)

-  *iso* ( optional)
-  *netmasks* (optional)
-  *gateway* (optional)
-  *dns* (optional) Dns servers
-  *domain* (optional) Dns search domain
-  *vnc* Defaults to false (use spice instead)
-  *cloudinit* Defaults to true
-  *start* Defaults to true
-  *keys* (optional). Array of public keys to inject
-  *cmds* (optional). Array of commands to run
-  *profile* name of one of your profile. Only checked in plan file
-  *scripts* array of paths of custom script to inject with cloudinit.
   Note that it will override cmds part. You can either specify full
   paths or relative to where you're running kcli. Only checked in
   profile or plan file

ansible support
---------------

you can check klist.py in the extra directory and use it as a dynamic
inventory for ansible.

The script uses sames conf as kcli ( and as such defaults to local
hypervisor if no configuration file is found)

vm will be grouped by plan, or put in the kvirt group if they dont
belong to any plan.

Interesting thing is that the script will try to guess the type of vm
based on its template, if present, and populate ansible\_user
accordingly

Try it with:

::

    python extra/klist.py --list

    ansible all -i extra/klist.py -m ping

Additionally, there s an ansible kcli/kvirt module under extras, with a
sample playbook

testing
-------

basic testing can be run with pytest. If using a remote hypervisor, you
ll want to set the *KVIRT\_HOST* and *KVIRT\_USER* environment variables
so that it points to your host with the corresponding user.

issues found with cloud images
------------------------------

-  for ubuntu latest images ( xenial), one needs to use something like
   guestfish to edit /boot/grub/grub.cfg and /etc/default/grub and
   remove console=ttyS0 from it.
-  Also note that you need to install python-simplejson ( actually
   bringing python2.7) to allow ansible to work on ubuntu
-  debian images are freezing. rebooting fixes the issue but as such
   cloudinit doesnt get applied...

Problems?
---------

Send me a mail at karimboumedhel@gmail.com !

Mac Fly!!!

karmab

.. |Build Status| image:: https://travis-ci.org/karmab/kcli.svg?branch=master
   :target: https://travis-ci.org/karmab/kcli
.. |Pypi| image:: http://img.shields.io/pypi/v/kcli.svg
   :target: https://pypi.python.org/pypi/kcli/
.. |asciicast| image:: https://asciinema.org/a/3p0cn60p0c0j9wd3hzyrs4m0f.png
   :target: https://asciinema.org/a/3p0cn60p0c0j9wd3hzyrs4m0f?autoplay=1
