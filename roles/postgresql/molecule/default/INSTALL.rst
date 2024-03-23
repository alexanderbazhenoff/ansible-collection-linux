##############
 Requirements
##############

Ansible `molecule requirements`_ page available at:

.. _molecule requirements: <https://molecule.readthedocs.io/en/latest/installation.html>`

Please refer to the `Virtual environment`_ documentation for
installation best practices. If not using a virtual environment, please
consider passing the widely recommended `'--user' flag`_ when invoking
``pip``.

.. _'--user' flag: https://packaging.python.org/tutorials/installing-packages/#installing-to-the-user-site

.. _virtual environment: https://virtualenv.pypa.io/en/latest/

.. code:: bash

   python3 -m pip install molecule flake8
   python3 -m pip install --upgrade --user setuptools
   python3 -m pip install --user "molecule"
   python3 -m pip install --user "molecule[ansible-lint]"
   python3 -m pip install --user "molecule[docker]"
