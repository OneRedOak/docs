:title: python 
:description: Configuring yml file for python
:keywords: Python, pip, nosetests, mirrors

.. _langpython:

Python
======

This section helps you to configure the yml file for your python project.

**Choosing Python versions to test against**

- Tell us what your build environment is. This is an option setting and if ommitted Ubuntu 12.04 is used as a default.
    .. code-block:: bash
    
        # Build Environment
              build_environment: ubuntu1204

**Python versions for testing** :

- Set the appropriate language and the version number. You can test against multiple version for a single push by adding more entries. Python minions use ``python`` by default to set the version.
      .. code-block:: python
        
          # language setting
             language: python

          # version numbers, testing against two versions of node
            python:
             - "2.7"
             - "3.2"
             - "3.3"
	     - "pypy"	

- We support for different versions of python like 2.6, 2.7, 3.2, 3.3 and pypy.
 
- Install dependencies for your project using **install** key.
	.. code-block:: python

	     install: "pip install -r requirements.txt --use-mirrors"


- **Test scripts** : Use **script** key in shippable.yml file to specify what command to run tests with.
	.. code-block:: python

		# command to run tests
		script: nosetests
	

- Test against multiple versions of Django by setting it on **env** key and then install the required dependencies for it using install key.
	.. code-block:: python

		env:
 		 - DJANGO_VERSION=1.2.7
		 - DJANGO_VERSION=1.3.7
 		 - DJANGO_VERSION=1.4.10
		 - DJANGO_VERSION=1.5.5
		 - DJANGO_VERSION=1.6

		install:
  		 - pip install -q mock==0.8 Django==$DJANGO_VERSION 
  		 - pip install . --use-mirrors

.. note::
 We are setting multiple versions here which means a single push to repo will trigger multiple builds. 
