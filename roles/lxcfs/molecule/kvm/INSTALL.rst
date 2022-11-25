Requirements
============

Ansible `molecule requirements`_ page available at:

.. _molecule requirements: <https://molecule.readthedocs.io/en/latest/installation.html>`

Please refer to the `Virtual environment`_ documentation for installation best
practices. If not using a virtual environment, please consider passing the
widely recommended `'--user' flag`_ when invoking ``pip``.

.. _Virtual environment: https://virtualenv.pypa.io/en/latest/
.. _'--user' flag: https://packaging.python.org/tutorials/installing-packages/#installing-to-the-user-site

.. code-block:: bash

    python3 -m pip install molecule flake8 molecule-libvirt libvirt-python
    python3 -m pip install --upgrade --user setuptools
    python3 -m pip install --user "molecule"
    python3 -m pip install --user "molecule[ansible-lint]"
    python3 -m pip install --user "molecule[docker]"


For testing this role you should have:

- Cloud images URL(s), e.g. `centos 7 cloud images`_.

.. _`centos 7 cloud images`:

<https://cloud.centos.org/centos/7/images/>

- Add current user (gitlab-runner, etc) to 'libvirt' (on Debian) and 'qemu-libvirt' groups:

.. code-block:: bash

    sudo usermod -aG $(cat /etc/passwd | cut -d: -f1 | grep libvirt) $(whoami)

- Create folder for testing and set permissions:

.. code-block:: bash

    sudo mkdir /var/lib/libvirt/images/ansible
    sudo chown $(whoami):$(whoami) /var/lib/libvirt/images/ansible
    sudo chmod 775 /var/lib/libvirt/images/ansible

- Created virtual network on the host to connect with virtual machines. This role testing use 'default' virt-net. In

most cases default nat network already created, but you can create them again:

xml:

.. code-block:: xml

    <network>
      <name>default</name>
      <forward mode="nat"/>
      <domain name="default"/>
      <ip address="192.168.175.1" netmask="255.255.255.0">
        <dhcp>
          <range start="192.168.175.128" end="192.168.175.254"/>
        </dhcp>
      </ip>
    </network>

then run:

.. code-block:: bash

    sudo virsh net-create {xml_file}
    sudo virsh net-autostart default
    sudo virsh net-start default
    # (autostart and enable not for not for transient networks)

- To avoid an error 'cp: cannot open '/boot/vmlinuz-5.4.0-113-generic' for reading: Permission denied' of

virt-customize tools (e.g. on Ubuntu20.04 LTS) you should change:

.. code-block:: bash

    sudo chmod 644 /boot/$(ls /boot | grep "vmlinuz-$(uname -sr | awk '{print $2}')")

or use 'become' and 'become password'.
