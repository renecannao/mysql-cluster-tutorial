====
Setup your environment
====


Summary
~~~~

#. `Copy the content of the USB drive`_
#. `Download and install Virtualbox`_
#. `Download and install Vagrant`_


Copy the content of the USB drive
----

In the USB drive are present:

* VirtualBox binaries for most platforms (copy only what you need)

* Valgrant binaries for most platforms (copy only what you need)

* base vagrant box, Vagrantfile, MySQL Cluster binaries, config files, others

Copy all required filed in a local directory.



Download and install VirtualBox
----

During the tutorial we will use Virtual Machines running on Virtual Box.

VirtualBox can be downloaded and then installed from `here <https://www.virtualbox.org/wiki/Downloads>`_.



Download and install Vagrant
----

Vagrant allows the easy configuration and deployment of a VirtualBox environment. For this reason, Vagrant will be used to configure and deploy the Virtual Machine used in the tutorial.

Vagrant can be downloaded and then installed from `here <http://downloads.vagrantup.com/>`_.

For Vagrant to work properly you need to have the latest *VirtualBox Guest Additions* . 

Load the base vagrant box::

  shell> vagrant box add ndbcluster /path/to/local/dir/lucid32.box

Start the VMs::
  
  shell> vagrant up

