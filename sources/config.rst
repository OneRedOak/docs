:title: Shippable Configuration
:description: Complete guide to how Shippable is setup
:keywords: shippable, signup, login, registration

.. _setup:

A bit deeper
============

Build Minions and Build configuration are two things that you should care the most about when using Shippable. The sections below talk about these in greater detail.


**Minions**
-----------

Minions are Docker based containers that run your builds and tests. You can think of them as Build VMs. Each minion runs one build at a time, so the more minions you have, the more concurrency you will get.  

We automatically create one minion for you when you first sign in to Shippable. Depending on your subscription, you can create more minions by going to Settings->Minions and clicking on the + sign.

Each minion starts from a base image and can be customized by specifying ``before_install`` scripts in the YML file. A minion can be configured to run any package, library, or service that your application requires. There are some preinstalled tools and services that you can use to customize your minions even further. 

Operating Systems
.................

All our Linux minions start from a vanilla base image from the Docker registry. We support all images as a starting point for your minion. Minions can be further customized by using the ``before_install`` and ``install`` tags in ``shippable.yml`` that is in the root of your code repository.

(Coming soon) Our Windows minions are based on AWS AMI for Windows 2012.

State Management
................

Shippable maintains the state of each minion between builds. We believe that build speed is very important, so reinstalling everything and cloning git repos every single time just doesn't make sense. 

However, shippable allows you to clear files and folders between builds using **reset_minion** tag. Configure your yml file as shown below:

.. code-block:: bash
 
   reset_minion: true

Before the build, we check for the tag **reset_minion** and if it exists, we will delete all the files and folders from the previous builds. You can also achieve this by adding **[reset_minion]** in the commit message.


Common Tools
............

A set of common tools are available on all minions. The following is a list of available tools -

- Latest release of Git repository
- apt installer
- Networking tools  
  
  - curl
  - wget
  - OpenSSL

- At least 1 version of (check out the language documentation for specific versions)
  
  - Ruby
  - Node
  - Python 
  - OpenJDK
  - PHP
  - Go
  - Clojure

- Services
  
  - MySQL
  - Postgre
  - SQLLite
  - MongoDB
  - Redis
  - ElasticSearch
  - Selenium Server
  - Neo4j
  - Cassandra
  - CouchDB
  - RethinkDB

- Headless browser testing tools

  - xfvb
  - PhantomJS

- Libraries

  - ImageMagik
  - OpenSSL

- JDK Versions

  - Oracle JDK 7u6 (oraclejdk7)
  - OpenJDK 7 (alias: openjdk7)
  - OpenJDK 6 (openjdk6)
  - Oracle JDK 8 EA (oraclejdk8)

- Build Tools

  - Maven 3
  - Gradle 1.9
  - Make
  - SBT 0.12.1

- Preinstalled PIP Packages

  - mock
  - nosetests
  - py.test

- Gems

  - Bundler
  - Rake
 
- Addons
  
  - Firefox
  - Custom hostname
  - PostgreSQL

----------

**Configuration**
------------------

This section is generic to all build environments and all languages. If you are looking for language specific tags, please refer to our language guides for more information.

``shippable.yml``
.................

We believe that developers need complete control of their build configuration. We also realize that most developers don't want to log into a UI to make changes every single time. 

While mulling over the best way to give developers control without asking them to go through our UI, we came across ``.travis.yml``, an open source initiative that created the basic framework for this very problem. Following the same paradigm, we ask you to have ``shippable.yml`` in the root of the repository you want to build. The structure of shippable.yml closely mimics travis since we see no reason to reinvent the wheel. We do have additional tags for added functionality, and these will become more numerous as we evolve Shippable. 

Since shippable.yml is a superset of ``.travis.yml`` , we support ``.travis.yml`` natively as well. So if you have a travis.yml in the root of your repo, we will read the config and set up your CI.

At a minimum, Shippable needs to have your language and build version specified in the yml. We will then default to the most common commands.

Build Flow
..........

When we receive a build trigger through a webhook or manual run, we execute the following steps - 

1. Clone/Pull the project from Github. This depends on whether the minion is in pristine state or not
2. ``cd`` into the workspace
3. Checkout the commit that is getting built
4. Run the ``before_install`` section. This is typically used to prep your minion and update any packages
5. Run ``install`` section to install any project specific libraries or packages
6. Run ``before_script`` section to create any folders and unzip files that might be needed for testing. Some users also restore DBs etc. here
7. Run the ``script`` command which runs build and all your tests
8. Run ``after_script`` command
9. Run either ``after_success`` or ``after_failure`` commands
10. Run ``before_archive`` command to copy files to ./shippable folder. Shippable will zips up all the files in this folder and makes the result available for download  
11. Run ``after_archive`` command to get the api access token to download the artifacts


The outcome of all the steps upto 7 determine the outcome of the build status. They need to return an exit code of ``0`` to be marked as success. Everything else is treated as a failure.


----------

**Other useful configs**
------------------------

Shippable uses Docker containers to provide you with isolation and a dedicated build environment. Our command sessions are not sticky throughout the build, but they are sticky within the same section of the build. For e.g. ``cd`` is sticky within the ``before_script`` tag of ``shippable.yml``

script
......
You can run any script file as part of your configuration, as long as it has a valid shebang command and the right ``chmod`` permissions. 

.. code-block:: python
        
        # script file 
        script: ./minions/do_something.sh 



command collections
...................
``shippable.yml`` supports collections under each tag. This is nothing more than YML functionality and we will run it one command at a time.

.. code-block:: python
        
  # collection scripts 
  script: 
   - ./minions/do_something.sh 
   - ./minions/do_something_else.sh 

In the example above, our minions will run ``./minions/do_something.sh`` and then run ``./minions/do_something-else.sh``. The only requirement is that all of these operations return a ``0`` exit code. Else the build will fail.


git submodules
..............
Shippable supports git submodules. This is a cool functionality of breaking your projects down into manageable chunks. We automatically initialize the ``.gitmodules`` file in the root of the repo. 

.. note::

  If you are using private repos, add the deploy keys so that our minion ssh keys are allowed to pull from the repo. This can be done via shippable.com

If its your own public repos then do this

.. code-block:: python
        
  # for public modules use
  git://github.com/someuser/somelibrary.git

  # for private modules use
  git@github.com:someuser/somelibrary.git

If you would like to turn submodules off completely -

.. code-block:: python
        
  # for public modules use
  git:
   submodules: false

api access token
.................

Once the build is finished, shippable will automatically zips up all the files available under ./shippable folder and you can download it using the **Download** button from the build details tab. You can also configure your yml file as shown below to get the api access token from console log to download the artifacts.

.. code-block:: python
 
 # move or copy files to shippable folder
   before_archive:
       - ls
       - mv  calculator.php  shippable/src
 
   after_archive:
     # To get the URL of the api access token
       - echo $SHIPPABLE_ARTIFACTS_URL
     # value of the below variable will be true if archive is successful else it will be false.
       - echo $ARTIFACTS_UPLOAD_SUCCESSFUL
 
.. note::  This URL is valid only for 20 minutes from the time build finishes off execution.



You can also configure your yml to trigger any other project's build using an api access token. For example, inorder to trigger build for project B, add the following line in any script section of project A's shippable.yml. 


.. code-block:: python

    curl -XPOST https://api.shippable.com/projects/<project_B_project_id>/build?token=$SHIPPABLE_API_TOKEN


Replace the <project_B_project_id> of the above line with project id provided in the console log of project B. $SHIPPABLE_API_TOKEN is unique to each build and we need this token to check for the authentication. If it is valid, then we will trigger the build for project B.    
 
  
common environment variables
.............................

You will have the following environment variables available to you for every build. You can use these in your scripts if required -

- BRANCH : Name of branch being built

- BUILD_NUMBER : Build number for current build

- BUILD_URL : Direct URL link to the Build Output

- COMMIT : Commit id that is being built and tested

- DEBIAN_FRONTEND : noninteractive

- JOB_ID : id of job in Shippable

- JRUBY_OPTS : --server -Dcext.enabled=false -Xcompile.invokedynamic=false

- LANG : en_US.UTF-8

- LC_ALL : en_US.UTF-8

- MERB_ENV : test

- PULL_REQUEST : Pull request id if the job is a pull request. If not, this will be set to 'none'

- RACK_ENV : test

- RAILS_ENV : test

- USER : shippable

- SHIPPABLE_ARTIFACTS_URL : URL to download artifacts

- ARTIFACTS_UPLOAD_SUCCESSFUL : Value of this variable will be true if archive is successful else this will be set as false.

- SHIPPABLE_API_TOKEN : Api access token for current build
 

user specified environment variables
.....................................

You can set your own environment variables in the yml. Every statement of this command will trigger a separate build with that specific version of the environment variables. 

.. code-block:: python
        
  # environment variable
  env:
   - FOO=foo BAR=bar
   - FOO=bar BAR=foo


.. note::

  Env variables can create an exponential number of builds when combined with ``jdk`` & ``rvm, node_js etc.`` i.e. it is multiplicative

In this setting **4 builds** are triggered

.. code-block:: python
        
  # npm builds
  node_js:
    - 0.10.24
    - 0.8.14
  env:
    - FOO=foo BAR=bar
    - FOO=bar BAR=foo

.. _secure_env_variables:

Secure environment variables
.............................

Shippable allows you to encrypt the environment variable definitions and keep your configurations private using **secure** tag. Go to settings -> Repositories -> click on the enabled project name -> and select Secure variables tab. Enter the env variable and its value in the text box as shown below. 

.. code-block:: python

    name=abc

Click on the encrypt button and copy the encrypted output string and add it to your yml file as shown below:


.. code-block:: python
   
   env:
     secure: <encrypted output>


To encrypt multiple environment variables and use them as part of a single build, enter the environment variable definitions in the text box as shown below 

.. code-block:: python

  name1="abc" name2="xyz"    

This will give you a single encrypted output that you can embed in your yml file.


You can also combine encrypted output and clear text environments using **global** tag. 

.. code-block:: python
 
   env:
     global:
       - FOO="bar"
       - secure: <encrypted output>


To encrypt multiple environment variables separately, configure your yml file as shown below: 

.. code-block:: python
  
  env:
    global:
      #encrypted output of first env variable
      - secure: <encrypted output> 
      #encrypted output of second env variable
      - secure: <encrypted output>
    matrix:
      #encrypted output of third env variable
      - secure: <encrypted output>


include & exclude branches
..........................

You can build specific branches or exclude them if needed. 

.. code-block:: python

  # exclude
  branches:
    except:
      - test1
      - experiment2

  # include
  branches:
    only:
      - stage
      - prod


build matrix
............

This is another powerful feature that Shippable has to offer. You can trigger multiple different test passes for a single code push. You might want to test against different versions of ruby, or different aspect ratios for your Selenium tests or best yet, just different jdk versions. You can do it all with Shippable's matrix build mechanism.

.. code-block:: python

  rvm:
    - 1.8.7 # (current default)
    - 1.9.2
    - 1.9.3
    - rbx
    - jruby
    - ruby-head
    - ree
  gemfile:
    - gemfiles/Gemfile.rails-2.3.x
    - gemfiles/Gemfile.rails-3.0.x
    - gemfiles/Gemfile.rails-3.1.x
    - gemfiles/Gemfile.rails-edge
  env:
    - ISOLATED=true
    - ISOLATED=false

The above example will fire 36 different builds for each push. Whoa! Need more minions?
 

**exclude**

It is also possible to exclude a specific version using exclude tag. Configure your yml file as shown below to exclude a specific version.

.. code-block:: python

   matrix:
     exclude:
       - rvm: 1.9.2
        


**include**

You can also configure your yml file to include entries into the matrix with include tag.

.. code-block:: python

   matrix:
     include:
       - rvm: 2.0.0
         gemfile: gemfiles/Gemfile.rails-3.0.x
         env: ISOLATED=false


**allow-failures**

Allowed failures are items in your build matrix that are allowed to fail without causing the entire build to be shown as failed. You can define allowed failures in the build matrix as follows:

.. code-block:: python

  matrix:
    allow_failures:
      - rvm: 1.9.3



----------

**Services**
-----------------
Shippable offers a host of pre-installed services to make it easy to run your builds. In addition to these you can install other services also by using the ``install`` tag of ``shippable.yml``. 

All the services are turned off by default and can be turned on by using the ``services:`` tag.

MongoDB
.......

.. code-block:: bash
  
  # Mongo binds to 127.0.0.1 by default
  services:
   - mongodb

Sample PHP code using `mongodb <https://github.com/Shippable/sample_php_mongo>`_ .


MySQL
.....

.. code-block:: bash
  
  # MySQL binds to 127.0.0.1 by default and is started on boot. Default username is shippable with no password
  # Create a DB as part of before script to use it

  before_script:
      - mysql -e 'create database myapp_test;'
                                 
Sample javascript code using `mysql <https://github.com/Shippable/sample_node_mysql>`_.


SQLite3
.......

SQLite is a software library that implements a self-contained, serverless, zero-configuration, transactional SQL database engine. So you can use SQLite, if you do not want to test your code behaviour with other databases.

Sample python code using `SQLite <https://github.com/Shippable/sample_python_sqllite>`_.


Elastic Search
..............

.. code-block:: bash

  # elastic search is on default port 9200
  services:
      - elasticsearch

Sample python code using `Elastic Search <https://github.com/Shippable/sample_python_elasticsearch>`_.

Memcached
..........

.. code-block:: bash

  # memcached runs on default port 11211
  services:
      - memcached

Sample python code using `Memcached <https://github.com/Shippable/sample_python_memcache>`_ .


Redis
.....

.. code-block:: bash

  # redis runs on default port 6379
  services:
      - redis


Sample python code using `Redis <https://github.com/Shippable/sample_python_redis>`_.


Neo4j
.....

.. code-block:: bash
 
 #neo4j runs on default port 7474
 services:
  - neo4j

Sample javascript code using `Neo4j <https://github.com/Shippable/sample_node_neo4j>`_ .

Cassandra
..........

.. code-block:: bash
 
 # cassandra binds to the default localhost 127.0.0.1 and is not started on boot. 
 services:
   - cassandra

Sample ruby code using `Cassandra <https://github.com/Shippable/sample_ruby_cassandra>`_ .

CouchDB
.........

.. code-block:: bash

 # couchdb binds to the default localhost 127.0.0.1 and runs on default port 5984. It is not started on boot.
 services:
   - couchdb

Sample ruby code using `CouchDB <https://github.com/Shippable/sample-ruby-couchdb/blob/master/shippable.yml>`_ .

RethinkDB
...........

.. code-block:: bash

 # rethinkdb binds to the default localhost 127.0.0.1 and is not started on boot.
 services:
   - rethinkdb

Sample javascript code using `RethinkDB <https://github.com/Shippable/sample-node-rethinkdb>`_.
 
RabbitMQ
.........

.. code-block:: bash

  # rabbitmq binds to 127.0.0.1 and is not started on boot. Default vhost "/", username "guest" and password "guest" can be used.
  services:
    - rabbitmq

Sample python code using `RabbitMQ <https://github.com/Shippable/sample_python_rabbitmq>`_ .


Selenium
.........

Selenium is not started on boot. You will have to enable it using **services** tag and start xvfb (X Virtual Framebuffer) on display port 99.0, so that all your test suites will run on the server without a display. Configure your yml file as shown below to start selenium on firefox.

.. code-block:: bash
   
     addons:
        firefox: "23.0"
     services:
       - selenium
     before_script:
       - "export DISPLAY=:99.0"
       - "/etc/init.d/xvfb start"
     after_script:
       - "/etc/init.d/xvfb stop"

     
Sample javascript code using `Selenium <https://github.com/Shippable/sample_node_selenium>`_ .


--------

**Addons**
----------

firefox
..........

We support different firefox versions like "18.0", "19.0", "20.0", "21.0", "22.0", "23.0", "24.0", "25.0", "26.0", "27.0", "28.0", "29.0". To select a specific firefox version, add the following to your shippable.yml file.

.. code-block:: python

	addons:
  	   firefox: "21.0"

custom host name
..................

You can also set up custom hostnames using the **hosts** addons. To set up the hostnames in /etc/hosts file, add the following to your shippable.yml file.
   
.. code-block:: python

        addons:
           hosts: 
    	    - google.com
            - asdf.com

PostgreSQL
...........

.. code-block:: bash

  # Postgre binds to 127.0.0.1 by default and is started on boot. Default username is "postgres" with no password
  # Create a DB as part of before script to use it

  before_script:
    - psql -c 'create database myapp_test;' -U postgres

Sample java code using `PostgreSQL <https://github.com/Shippable/sample_java_postgres>`_.

We support PostgreSQL 9.1, 9.2 and 9.3 versions and by default, version 9.2 is installed on our minions. Configure your yml file using **PostgreSQL** addons to select different versions. Add the following to your yml file to select the version 9.3.


.. code-block:: python

          addons:
           postgresql : "9.3"
  
PostGIS 2.1 packages are pre-installed in our minions along with the PostgreSQL versions 9.1, 9.2 and 9.3.


----------

**Test and Code Coverage visualization**
----------------------------------------
Test results
............
To set up test result visualization for a repository.

* Output test results to shippable/testresults folder. 
* Make sure test results are in junit format.

For example, here is the .yml file for a Python repo -

.. code-block:: bash

  before_script: mkdir -p shippable/testresults
  script:
    - nosetests python/sample.py --with-xunit --xunit-file=shippable/testresults/nosetests.xml

Examples for other languages can be found in our :ref:`Code Samples <samplesref>` .

Code coverage
.............
To set up code coverage result visualization for a repository.

* Output code coverage output to shippable/codecoverage folder. 
* Make sure code coverage output is in cobertura xml format.

For example, here is the .yml file for a Python repo -

.. code-block:: bash

  before_script: mkdir -p shippable/codecoverage
  script:
    - coverage run --branch python/sample.py
    - coverage xml -o shippable/codecoverage/coverage.xml python/sample.py

Examples for other languages can be found in our :ref:`Code Samples <samplesref>`.

----------

**Notifications**
-----------------
Shippable can notify you about the status of your build. If you want to get notified about the build status (success, failure or unstable), you need to follow the rules below to configure your yml file. Shippable will send the consolidated build reports in individual emails for matrix build projects. By default Shippable will send the email notifications to the last committer.


Email notification
..................


You can configure the email notification by specifying the recipients id in ``shippable.yml`` file.

.. code-block:: bash

  notifications:
      email:
          - exampleone@org.com
          - exampletwo@org.com


You can also specify when you want to get notified using change|always|never. Change means you want to be notified only when the build status changes on the given branch. Always and never mean you want to be notified always or never respectively.


.. code-block:: bash
 
  notifications:
       email:
           recipients:
               - exampleone@org.com
               - exampletwo@org.com
           on_success: change
           on_failure: always


If you do not want to get notified, you can configure email notifications to false.

.. code-block:: bash

  notifications:
     email: false


---------

**Continuous deployment**
-------------------------

Continuous deployment to Heroku
...............................

Heroku supports Ruby, Java, Python, Node.js, PHP, Clojure and Scala (with special support for Play Framework), so you can use these technologies to build and deploy apps on Heroku.
There are two methods of deploying your applications to Heroku: using Heroku toolbelt or plain git command only.

Without Heroku toolbelt
^^^^^^^^^^^^^^^^^^^^^^^

To be able to push your code to Heroku, you need to add SSH public key associated with your Shippable account to authorized keys in `Heroku Account Settings <https://dashboard.heroku.com/account>`_.
In Shippable, go to 'Settings' and choose 'Deployment key' tab. Copy the contents of the key and add it in 'SSH keys' section of Heroku settings.

Next, create your Heroku application using Web GUI or ``heroku`` command installed on your workstation.

* Go to your app's settings page.
* In the application info pane (that is also displayed at the end of the application creation process) you will see 'Git URL'.
* Just use it to push the code in ``after_success`` step of Shippable build:

.. code-block:: bash

  env:
    global:
      - APP_NAME=shroudd-headland-1758

  after_success :
    - git push -f git@heroku.com:$APP_NAME.git master

Full sample of deploying PHP+MySQL application to Heroku without using toolbelt can be found on `our GitHub account <https://github.com/Shippable/sample-php-mysql-heroku/tree/without-toolbelt>`_.

.. note::

  If you happen to build other branches than ``master``, please see :ref:`heroku_other_branches` for details.

With Heroku toolbelt
^^^^^^^^^^^^^^^^^^^^

In some circumstances, you may choose to use Heroku toolbelt to deploy your application. For example, you may want to execute Rake commands straight from your Shippable build (like ``heroku run rake db:migrate``).

To use Heroku toolbelt, you first need to obtain API key for your account. Go to your `account settings <https://dashboard.heroku.com/account>`_ and copy it from 'API Key' section.
It is recommended to save your access key as secret in Shippable, as is discussed in :ref:`secure_env_variables`. Encrypt your variable as ``HEROKU_API_KEY=<your key here>`` and paste the encrypted secret in ``shippable.yml`` as follows:

.. code-block:: bash

  env:
    global:
      - APP_NAME=shroudd-headland-1758
      - secure: MRuHkLbL9HPkJPU5lzkKM1+NOq1S5RrhxEyhJkk60xxYiF7DMzydiBN8oFBjWrSmyGeGRuEC22a0I5ItobdWVszfcJCaXHwtfKzfGOUdKuyCnDgvojXhv/jrBvULyLK6zsLw3b8NMxdnwNsHqSPm19qW/EIGEl9Zv/637Igos69z9aT7+xrEG013+6HtKYb8RHm+iPSNsFoBi/RSAHYuM1eLTZWG2WAkjgzZaYmrHCgNwVmk+HOGR+TOWN7Iu5lrjyvC1XDCQrOvo1hZI30cd9OqJ5aadFm3exQpNhI4I7AgOnCbK3NoWNc/GAnqKXCvsaIQ80Jd/uLIOVyMjD6Xmg==

.. note::

  If your build times out during ``after_success`` step, please double check that you correctly defined ``HEROKU_API_KEY`` variable.
  If no key is supplied, Heroku toolbelt will switch to an interactive mode, prompting for the username and causing the build to 'hang'.

Then, install the toolbelt in ``before_install`` step (``which heroku`` is for skipping this step if the tools are already installed):

.. code-block:: bash

  before_install:
    - which heroku || wget -qO- https://toolbelt.heroku.com/install-ubuntu.sh | sh

Next, create your Heroku application using Web GUI or ``heroku`` command installed on your workstation. Then, add the following ``after_success`` step to your Shippable build: 

.. code-block:: bash

  after_success:
    - test -f ~/.ssh/id_rsa.heroku || ssh-keygen -y -f ~/.ssh/id_rsa > ~/.ssh/id_rsa.heroku && heroku keys:add ~/.ssh/id_rsa.heroku
    - git remote -v | grep ^heroku || heroku git:remote --app $APP_NAME
    - git push -f heroku master

* First we generate public SSH key out of the private one to a file with a custom name. We then authorize this key with Heroku. Using custom name for the file allows us to skip this step on subsequent builds.
  Please note that we need to use ``test -f ...`` instead of ``[ -f ... ]`` here, as the latter would be interpreted by YAML parser
* Then, we make sure ``heroku`` remote is added to the local git repository
* Finally, we push the code to Heroku

Please refer to the sections below for language-specific details of configuring Heroku builds.

.. note::

  If you happen to build other branches than ``master``, please see :ref:`heroku_other_branches` for details.

.. _heroku_other_branches:

Deploying from branches other than master
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Heroku always deploys contents of ``master`` branch, so if you happen to deploy code from other branches, `the Heroku documentation <https://devcenter.heroku.com/articles/git#deploying-code>`_
instructs you to use the following syntax:

.. code-block:: bash

  - git push heroku yourbranch:master

However, Shippable sets its local repository in such a way that the current branch is always seen as ``master``, no matter how it is called in your remote repository. For this reason, make sure
to always use only remote branch name when deploying to Heroku:

.. code-block:: bash

  - git push heroku master

Using ClearDB MySQL database
^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Heroku passes ClearDB MySQL connection details as an environment variable called ``CLEARDB_DATABASE_URL`` containing connection URL.
To mock it with the test database during build, add the following environment variable in your ``shippable.yml`` config:

.. code-block:: bash

  env:
    global:
      - CLEARDB_DATABASE_URL=mysql://shippable@127.0.0.1:3306/test?reconnect=true

Then, in your application you need to retrieve and parse the url. For example, in PHP:

.. code-block:: php

    $url = parse_url(getenv("CLEARDB_DATABASE_URL"));
    $host = $url["host"];
    $username = $url["user"];
    $password = array_key_exists("pass", $url) ? $url["pass"] : "";
    $db = substr($url["path"], 1);

    $con = mysqli_connect($host, $username, $password, $db);

Please refer to `Heroku docs <https://devcenter.heroku.com/articles/cleardb>`_ for details on how to fetch and parse the url in different programming languages.
Full sample of deploying PHP+MySQL application to Heroku (using Heroku toolbelt) can be found on `our GitHub account <https://github.com/Shippable/sample-php-mysql-heroku>`_.

Using Heroku Postgres with Ruby on Rails
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Configuring Ruby on Rails application to work with Postgres on Heroku is really simple, thanks to Heroku doing all heavy-lifting related to setting up the connection.
When Heroku detects that the application you deploy is using Ruby on Rails, it will overwrite ``config/database.yml`` file with correct production configuration.

On Shippable, Postgres in version 9.3 is started by default during minion boot. To use different version of Postgres, please refer to the
dedicated section on PostgreSQL configuration.

All we need to do is to create a database in the ``before_script`` step:

.. code-block:: bash

  before_script: 
    - mkdir -p shippable/testresults
    - mkdir -p shippable/codecoverage
    - psql -c 'create database "sample-rubyonrails-postgres-heroku_test";' -U postgres

And then include its name in ``config/database.yml`` file that is stored in the repository (username and password do not need to be configured):

.. code-block:: bash

  test:
    <<: *default
  database: sample-rubyonrails-postgres-heroku_test

The last thing to do is to add ``pg`` to your ``Gemfile``. Note that it will be done automatically if you create your rails app with ``--database=postgresql`` option.
See `our sample Ruby on Rails Heroku application <https://github.com/Shippable/sample-rubyonrails-postgres-heroku>`_ for details.

Test and coverage reports for Ruby on Rails
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

In Rails 4 and Ruby 1.9+, the built-in test framework is based on Minitest.
To enable Shippable-compatible reporting of test and coverage reports, we need to add the following gems to the ``Gemfile``:

.. code-block:: bash

  gem 'simplecov'
  gem 'simplecov-csv'
  gem 'minitest-reporters'

Then, add the following snippet at the beginning of the ``test/test_helper.rb`` file:

.. code-block:: bash

  require 'minitest/reporters'
  require 'simplecov'
  require 'simplecov-csv'
  SimpleCov.formatter = SimpleCov::Formatter::CSVFormatter
  SimpleCov.coverage_dir(ENV["COVERAGE_REPORTS"])
  SimpleCov.start

  MiniTest::Reporters.use! [MiniTest::Reporters::DefaultReporter.new,
                            MiniTest::Reporters::JUnitReporter.new(ENV["CI_REPORTS"])]

Finally, we need to add environment variables with the locations for the reporter results:

.. code-block:: bash

  env:
    global:
      - CI_REPORTS=shippable/testresults COVERAGE_REPORTS=shippable/codecoverage

See `our sample Ruby on Rails Heroku application <https://github.com/Shippable/sample-rubyonrails-postgres-heroku>`_ for details.

General information on using MongoDB
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

You have two addons to choose from when using MongoDB on Heroku: MongoLab and MongoHQ. Setup for both of them with Shippable is the same.
The only difference is the name of the environment variable that contains connection details:

* For MongoLab it is called ``MONGOLAB_URI``
* For MongoHQ the name of the variable is ``MONGOHQ_URL``

Examples below use MongoLab for consistency, but adapting them to MongoHQ is as simple as substituting all occurrences of this variable.

To start using MongoDB, first add the addon of your choice to your Heroku application. 

In your ``shippable.yml`` you first need to tell Shippable to provide your build with MongoDB service.
Then, provide mock connection URL to be used by your tests. On Shippable, MongoDB is accessed without providing user nor password.

.. code-block:: bash

  services:
     - mongodb

  env:
    global:
      - APP_NAME=rocky-wave-3011
      - MONGOLAB_URI=mongodb://localhost/test

Then proceed to configure your application as is outlined in per-language guides below.

Using MongoDB with PHP
^^^^^^^^^^^^^^^^^^^^^^

First, activate the official Mongo driver extension in ``php.ini`` on Shippable minion, as is explained in the documentation on PHP extensions:

.. code-block:: bash

  before_script: 
    - mkdir -p shippable/testresults
    - mkdir -p shippable/codecoverage
    - echo "extension=mongo.so" >> ~/.phpenv/versions/$(phpenv version-name)/etc/php.ini

Then, tell Heroku to enable the extension as well by providing the following ``composer.json`` file in the root of your repository:

.. code-block:: json

  {
    "require": {
      "ext-mongo": "*"
    }
  }

Finally, you can connect to the database with the code as follows: 

.. code-block:: php

  $mongoUrl = getenv("MONGOLAB_URI");
  $dbName = substr(parse_url($mongoUrl)["path"], 1);
  $this->mongo = new Mongo($mongoUrl);
  $this->scores = $this->mongo->{$dbName}->scores;

Please note that we need to parse the URL to the instance to extract the database name, as the Mongo driver expects that the database is selected by accessing property named
the same as the database, which is demonstrated in the last line of the snippet above.

Full sample of deploying PHP+MongoDB application to Heroku (using Heroku toolbelt) can be found on `our GitHub account <https://github.com/Shippable/sample-php-mongo-heroku>`_.

Using MongoDB with Python
^^^^^^^^^^^^^^^^^^^^^^^^^

First, create file called ``Procfile`` that will tell Heroku how to launch your Python application. For example, if you use Flask and create the application under name ``application``
in ``hello.py``

.. code-block:: bash

  web: gunicorn hello:application

Next, declare that your application depends on MongoDB official driver. Heroku requires ``requirements.txt`` file in the root of your repository that may be generated with ``pip freeze`` command.
For our (Flask) example, we will use file with contents as follows:

.. code-block:: bash

  nose
  coverage
  gunicorn
  Flask
  pymongo

Finally, you can connect to the database with the following code:

.. code-block:: python

  mongo_url = os.environ['MONGOLAB_URI']
  db_name = urlparse.urlparse(mongo_url).path[1:]
  client = MongoClient(mongo_url)
  self.db = client[db_name]

Please note that we need to parse the URL to the instance to extract the database name, as the Python Mongo driver follows the convention of accessing the database by property with
the same as the database, which is demonstrated in the last line of the snippet above.

Full sample of deploying Python+MongoDB application to Heroku (using Heroku toolbelt) can be found on `our GitHub account <https://github.com/Shippable/sample-python-mongo-heroku>`_.

Using MongoDB with Ruby
^^^^^^^^^^^^^^^^^^^^^^^

First, create a file called ``Procfile`` that will tell Heroku how to launch your application. For example, if you use Sinatra and your application entry point is located in file
called ``helloworld.rb``:

.. code-block:: bash

  web: bundle exec ruby helloworld.rb -p $PORT

Next, declare your dependencies in ``Gemfile``. For example, if using `Mongoid <http://mongoid.org/>`_ to access the database:

.. code-block:: bash

  source "https://rubygems.org"

  gem "rspec"
  gem "simplecov"
  gem "simplecov-csv"
  gem "rspec_junit_formatter"
  gem "sinatra"
  gem "mongoid"

To separate Mongoid configuration from your code, create YAML file (e.g. called ``mongoid.yml``):

.. code-block:: yaml

  production:
    sessions:
      default:
        uri: <%= ENV['MONGOLAB_URI'] %>

Next, use this file to connect to the database in your application:

.. code-block:: ruby

  require 'mongoid'
  Mongoid.load!('mongoid.yml', :production)

You can also execute Rake tasks in your ``after_success`` step using Heroku toolbelt. For example, to run database migrations at the end of the build:

.. code-block:: bash

  after_success:
    - test -f ~/.ssh/id_rsa.heroku || ssh-keygen -y -f ~/.ssh/id_rsa > ~/.ssh/id_rsa.heroku && heroku keys:add ~/.ssh/id_rsa.heroku
    - git remote -v | grep ^heroku || heroku git:remote --app $APP_NAME
    - git push -f heroku master
    - heroku run rake db:migrate

Full sample of deploying Sinatra+MongoDB application to Heroku (using Heroku toolbelt) can be found on `our GitHub account <https://github.com/Shippable/sample-ruby-mongo-heroku>`_.

Using MongoDB with Node.js
^^^^^^^^^^^^^^^^^^^^^^^^^^

First, create file called ``Procfile`` that will tell Heroku how to launch your Node.js application. For example, if you use Express and define your routes in file called ``app.js``:

.. code-block:: bash

  web: node app.js

Next, declare that your application depends on Mongoose (or other library of your choice). Heroku will read your ``package.json`` file:

.. code-block:: json

  ...
  "dependencies": {
    "express": "~4.2.0",
    "mongoose": "^3.8.12",
    "when": "~3.2.3"
  },

Finally, you can connect to the database with the following code:

.. code-block:: javascript

  var mongoose = require('mongoose');
  mongoose.connect(process.env.MONGOLAB_URI);

Full sample of deploying Express+MongoDB application to Heroku (using Heroku toolbelt) can be found on `our GitHub account <https://github.com/Shippable/sample-nodejs-mongo-heroku>`_.

Continuous deployment to Amazon Elastic Beanstalk
.................................................

Amazon Elastic Beanstalk features predefined runtime environments for Java, Node.js, PHP, Python and Ruby, so it is possible to configure Shippable minions to automatically deploy applications targeting these environments. Moreover, Elastic Beanstalk also support defining custom runtime environments via Docker containers, giving the developer full flexibility in the configuration of technology stack. However, as the standard, pre-packaged environments are far easier to set up and cover nearly all languages currently supported by Shippable (except for Go), we will concentrate on how to integrate with Amazon Elastic Beanstalk using the former option.

To interact with Elastic Beanstalk, one needs to use command line tools supplied by Amazon, from which the most commonly used is ``eb`` command. These tools must be available to Shippable in order to perform deployment. The easiest way of doing so is to download them in ``before_install`` step:

.. code-block :: bash

  env:
    global:
      - EB_TOOLS_DIR=/tmp/eb_tools EB_VERSION=AWS-ElasticBeanstalk-CLI-2.6.3 EB_TOOLS=$EB_TOOLS_DIR/$EB_VERSION

  before_install:
    - if [ ! -e $EB_TOOLS ]; then wget -q -O /tmp/eb.zip https://s3.amazonaws.com/elasticbeanstalk/cli/$EB_VERSION.zip && mkdir -p $EB_TOOLS_DIR && unzip /tmp/eb.zip -d $EB_TOOLS_DIR; fi

You also need to obtain Access Key to connect ``eb`` tool with Elastic Beanstalk API. Please refer to `this documentation <http://docs.aws.amazon.com/general/latest/gr/getting-aws-sec-creds.html>`_ for details on obtaining the keys. It is recommended to save your access key as secret in Shippable, as is discussed in :ref:`secure_env_variables`. To use code from this tutorial, store the secret access key variable as ``AWSSecretKey``. It is safe to keep your access key id in plain text.

After having the basic setup done, it is time to create an application in Elastic Beanstalk. You can use Web GUI for this task, by going to the main page in the Elastic Beanstalk console, and then choosing 'Create a New Application' button in the sidebar. After entering name for application, proceed to define an environment. When you have the environment ready, create a file in your repository called ``config`` with your settings, where ``DevToolsEndpoint`` is based on the AWS region you are using:

.. code-block :: bash

  [global]
  ApplicationName=bean-test
  DevToolsEndpoint=git.elasticbeanstalk.us-west-2.amazonaws.com
  EnvironmentName=bean-env
  Region=us-west-2
  ServiceEndpoint=https://elasticbeanstalk.us-west-2.amazonaws.com

In the runtime environment, RDS database connection details are injected by Elastic Beanstalk as environment variables. It is secure and convenient, as we do not need to store them in any other place. However, during tests on Shippable, we need to supply the same variables with values correct for Shippable minion in ``shippable.yml`` (please note the ``secure`` definition for our AWS access key):

.. code-block :: bash

  env:
    global:
      - EB_TOOLS_DIR=/tmp/eb_tools EB_VERSION=AWS-ElasticBeanstalk-CLI-2.6.3 EB_TOOLS=$EB_TOOLS_DIR/$EB_VERSION
      - RDS_HOSTNAME=127.0.0.1 RDS_USERNAME=shippable RDS_PASSWORD="" RDS_DB_NAME=test RDS_PORT=3306
      - secure: K7qw2XSFBaW+zEzrs0ODKMQq/Bo9AZqGotCXc50fao+et6WaxEmedlK//MO9JozmPdcxDRq5k8A0pmjTLsMLstkh7PUFLu3Z6xowU2OhMyjQ0pS2J8Hw16SgZ9n2EW+3cps4dIijEzOwjA0Yx5rTOC7F9N8nvr/1l4Yp4i11qgW08cefEKuwiF/ypgrkK5BYJyJreZOEJt3lJ6/aXyXxPPl3X3Z+L+ca9mQmTN6q1wnlEcYLDU5EJtkk87KtOfVyoi/+aOFh49eDpwStSD4zDnygia8eAnCGK/p0XGFJCAwWK1nnFY7aklJrvElD+V/2lbr14TwF0rhmiba6Y6ylnw==

Then, we need to add some steps to ``shippable.yml`` to update ``eb`` configuration and then launch it after successful build. We are invoking ``AWSDevTools-RepositorySetup.sh`` to configure git-based workflow for Elastic Beanstalk deployment (for instance, this command adds git remotes pointing to AWS endpoints).  Remember to replace value for ``AWSAccessKeyId`` with the one downloaded from your AWS Management Console:

.. code-block :: bash

  before_script: 
    - mkdir -p ~/.elasticbeanstalk
    - echo 'AWSAccessKeyId=AKIAJSZ63DT' >> ~/.elasticbeanstalk/aws_credential_file
    - echo 'AWSSecretKey='$AWSSecretKey >> ~/.elasticbeanstalk/aws_credential_file

  script:
    - mkdir -p .elasticbeanstalk
    - cp config .elasticbeanstalk/

  after_success :
    - $EB_TOOLS/AWSDevTools/Linux/AWSDevTools-RepositorySetup.sh
    - export PATH=$PATH:$EB_TOOLS/eb/linux/python2.7/ && virtualenv ve && source ve/bin/activate && pip install boto==2.14.0 && eb push

Finally, we can connect to the database using environment variables as defined above.

PHP
^^^

.. code-block :: php

  $con = mysqli_connect(
    $_SERVER['RDS_HOSTNAME'],
    $_SERVER['RDS_USERNAME'],
    $_SERVER['RDS_PASSWORD'],
    $_SERVER['RDS_DB_NAME'],
    $_SERVER['RDS_PORT']
  );            

Elastic Beanstalk serves your repository root as the document root of the webserver, so e.g. ``index.php`` file will be interpreted when you access the root context of your Elastic Beanstalk application. 
See full sample PHP code using Elastic Beanstalk available `on GitHub <https://github.com/Shippable/sample-php-mysql-beanstalk>`_.

Ruby
^^^^

.. code-block :: ruby

  con = Mysql2::Client.new(
    :host => ENV['RDS_HOSTNAME'],
    :username => ENV['RDS_USERNAME'],
    :password => ENV['RDS_PASSWORD'],
    :port => ENV['RDS_PORT'],
    :database => ENV['RDS_DB_NAME']
  )

Elastic Beanstalk runtime expects that the entry point of your application will be found in ``config.ru`` file. See `Amazon documentation <http://docs.aws.amazon.com/elasticbeanstalk/latest/dg/create_deploy_Ruby_sinatra.html>`_ for details.
Full sample Ruby code using Sinatra and MySQL on Elastic Beanstalk is available `on GitHub <https://github.com/Shippable/sample-ruby-mysql-beanstalk>`_.

Python
^^^^^^

.. code-block :: python

  self.db = MySQLdb.connect(
    host = os.environ['RDS_HOSTNAME'],
    user = os.environ['RDS_USERNAME'],
    passwd = os.environ['RDS_PASSWORD'],
    port = int(os.environ['RDS_PORT']),
    db = os.environ['RDS_DB_NAME'])

As Python projects are already run in ``virtualenv`` on Shippable minions, change the ``after_success`` step from the end of the general description above to:

.. code-block :: python

  after_success:
    - $EB_TOOLS/AWSDevTools/Linux/AWSDevTools-RepositorySetup.sh
    - export PATH=$PATH:$EB_TOOLS/eb/linux/python2.7/ && pip install boto==2.14.0 && eb push

Elastic Beanstalk runtime expects that the entry point of your application will be found in ``application.py`` file. See `Amazon documentation <http://docs.aws.amazon.com/elasticbeanstalk/latest/dg/create_deploy_Python_flask.html>`_ for details.
Full sample Python code using Flask and MySQL on Elastic Beanstalk is available `on GitHub <https://github.com/Shippable/sample-python-mysql-beanstalk>`_.

Node.js
^^^^^^^

.. code-block :: javascript

  var connection = mysql.createConnection({
    host: process.env.RDS_HOSTNAME,
    port: process.env.RDS_PORT,
    user: process.env.RDS_USERNAME,
    password: process.env.RDS_PASSWORD,
    database: process.env.RDS_DB_NAME
  });

Elastic Beanstalk expects that the entry point of your application will be found in `app.js` or `server.js` file in the repository root. See `Amazon documentation <http://docs.aws.amazon.com/elasticbeanstalk/latest/dg/create_deploy_nodejs_express.html>`_ for details.
Full sample Node.js code using Express and MySQL on Elastic Beanstalk is available `on GitHub <https://github.com/Shippable/sample-nodejs-mysql-beanstalk>`_.

Java
^^^^

For JVM, the connection setting are passed as system properties, rather than environment variables:

.. code-block :: java

  private final String dbName = System.getProperty("RDS_DB_NAME"); 
  private final String userName = System.getProperty("RDS_USERNAME"); 
  private final String password = System.getProperty("RDS_PASSWORD"); 
  private final String hostname = System.getProperty("RDS_HOSTNAME");
  private final int port = Integer.parseInt(System.getProperty("RDS_PORT"));
  private final String databaseUrl = "jdbc:mysql://" + hostname + ":" + port + "/" + dbName;

Because of this difference, dummy connection details for Shippable environment need to be passed as arguments during JVM invocation. Here is an example for Maven:

.. code-block :: bash

  script:
    - mkdir -p .elasticbeanstalk
    - cp config .elasticbeanstalk/
    - mvn clean cobertura:cobertura
    - mvn test -DRDS_PORT=3306 -DRDS_DB_NAME=test -DRDS_HOSTNAME=localhost -DRDS_PASSWORD= -DRDS_USERNAME=shippable

Finally, Elastic Beanstalk `expects exploded WAR <https://forums.aws.amazon.com/thread.jspa?messageID=329550>`_ in the root of the repository, so we need to copy and commit its contents as the final build step, prior to the deployment:

.. code-block :: bash

  after_success:
    - mvn compile war:exploded
    - cp -r target/App/* ./
    - git add -f META-INF WEB-INF
    - git commit -m "Deploy"
    - $EB_TOOLS/AWSDevTools/Linux/AWSDevTools-RepositorySetup.sh
    - export PATH=$PATH:$EB_TOOLS/eb/linux/python2.7/ && pip install boto==2.14.0 && eb push

See the full sample of Java web application featuring MySQL connection `on GitHub <https://github.com/Shippable/sample-java-mysql-beanstalk>`_ for details.

Scala
^^^^^

Scala deployment is very similar to the one described for Java above. First, RDS connection details need to be obtained from system properties, rather then environment variables.
Here is an example for `Slick <http://slick.typesafe.com/>`_:

.. code-block :: scala

  val dbName = System.getProperty("RDS_DB_NAME")
  val userName = System.getProperty("RDS_USERNAME")
  val password = System.getProperty("RDS_PASSWORD")
  val hostname = System.getProperty("RDS_HOSTNAME")
  val port = Integer.parseInt(System.getProperty("RDS_PORT"))
  val databaseUrl = s"jdbc:mysql://${hostname}:${port}/${dbName}"

  def connect = Database.forURL(
    url = databaseUrl, user = userName, password = password, driver = "com.mysql.jdbc.Driver")

Because of this difference, dummy connection details for Shippable environment need to be passed as arguments during JVM invocation. Here is an example for ``sbt``
(please note copying the coverage results, as `sbt-scoverage <https://github.com/scoverage/sbt-scoverage>`_ does not allow customizing the path via options):

.. code-block :: bash

  script:
    - mkdir -p .elasticbeanstalk
    - cp config .elasticbeanstalk/
    - sbt -DRDS_PORT=3306 -DRDS_DB_NAME=test -DRDS_HOSTNAME=localhost -DRDS_PASSWORD= -DRDS_USERNAME=shippable scoverage:test
    - cp target/scala-2.10/coverage-report/cobertura.xml shippable/codecoverage/coverage.xml

Finally, Elastic Beanstalk `expects exploded WAR <https://forums.aws.amazon.com/thread.jspa?messageID=329550>`_ in the root of the repository, so we need to copy and commit its contents as the final build step, prior to the deployment:

.. code-block :: bash

  after_success:
    - sbt package
    - unzip "target/scala-2.10/*.war" -d ./
    - git add -f META-INF WEB-INF
    - git commit -m "Deploy"
    - $EB_TOOLS/AWSDevTools/Linux/AWSDevTools-RepositorySetup.sh
    - export PATH=$PATH:$EB_TOOLS/eb/linux/python2.7/ && virtualenv ve && source ve/bin/activate && pip install boto==2.14.0 && eb push

See the full sample of Scalatra+Slick web application featuring MySQL connection `on GitHub <https://github.com/Shippable/sample-scala-mysql-beanstalk>`_ for details.

Continuous deployment to AWS OpsWorks
.....................................

AWS OpsWorks is a new PaaS offering from Amazon that targets advanced IT administrators and DevOps, providing them with more flexibility in defining runtime environments of their applications.
In OpsWorks, instances are arranged in so called layers, which in turn form stacks. Please refer to `the AWS documentation <http://docs.aws.amazon.com/opsworks/latest/userguide/gettingstarted.html>`_ for details.

OpsWorks allows provisioning instances with custom Chef recipes, which means unconstrained range of technologies that may be used on this platform.
Predefined Chef cookbooks are available for PHP, Ruby on Rails, Node.js and Java.

OpsWorks deployment process has slightly different nature than the one for Heroku or Amazon Elastic Beanstalk. While the former are 'push-based', meaning that the deployment is done by sending the build artifacts
to the platform, with OpsWorks you configure the service to pull the code and artifacts from a predefined resource.

This is done during definition of your application on OpsWorks, by entering URL for the repository.
Please note that for public access (without adding an SSH key), you need to use appropriate protocol for the endpoint, for example ``https://gihub.com/Shippable/sample-php-mysql-opsworks.git`` or ``git://gihub.com/Shippable/sample-php-mysql-opsworks.git``,
instead of SSH URL, such as ``github.com:Shippable/sample-php-mysql-opsworks.git``.

.. note::

  During our tests, some git commands (like ``ls-remote``) timed out for 'public' URLs on GitHub. This problem does not occur for SSH access, so you may need to create
  a SSH key for public repositories as well. To do so, execute ``ssh-keygen -f opsworks`` on your workstation and save the resulting files (``opsworks`` and ``opsworks.pub``)
  in a safe place. Then, add the contents of ``opsworks.pub`` to Deployment Keys in your GitHub repository settings. Next, paste the contents of ``opsworks`` file in SSH key
  box in Application definition in OpsWorks admin panel.

To integrate Shippable with OpsWorks, first define the stack, layers, instances and application as outlined in the AWS documentation.
We will use `AWS CLI tool <http://docs.aws.amazon.com/cli/latest/userguide/cli-chap-welcome.html>`_ to invoke deployments for your application.
In order for this to work, we need to provide the tools with your AWS access keys to authenticate with the AWS endpoint:

* Please refer to `this documentation <http://docs.aws.amazon.com/general/latest/gr/getting-aws-sec-creds.html>`_ for details on obtaining the keys. 
* Then, encrypt the secret key as discussed in :ref:`secure_env_variables`. Use ``AWS_SECRET_ACCESS_KEY`` as name for the secure variable (i.e. add ``AWS_SECRET_ACCESS_KEY=<your secret key here>`` in Shippable settings panel).
* Next, add the secret along with your key id as environment variables in ``shippable.yml`` (please note that name of the variable matters):

.. code-block :: bash

  env:
    global:
      - AWS_ACCESS_KEY_ID=AKIAJSZ63DTL3Z7KLVEQ
      - secure: KRaEGMHtRkYxCmWfvHIEkyfoA/+9EWHcoi1CIoIqXrvsF/ILmVVr0jC7X8u7FdfAiXTqn3jYGtLc5mgo5KXe/8zSLtygCr9U1SKJfwCgsw1INENlJiUraHCQqnnty0b3rsTfoetBnnY0yFIl2g+FUm3A57VnGXH/sTcpDZSqHfjCXivptWrSzE9s4W7+pu4vP+9xLh0sTC9IQNcqQ15L7evM2RPeNNv8dQ+DMdf48915M91rnPkxGjxfebAIbIx1SIhR1ur4rEk2pV4LOHo4ny3sasWyqvA49p1xItnGnpQMWGUAzkr24ggOiy3J5FnL8A9oIkf49RtfK1Z2F0EryA==

Finally, we can install and invoke AWS CLI tools to invoke deployment command in ``after_success`` step (application configuration settings were extracted to environment variables for readability):

.. code-block :: bash

  env:
    global:
      - AWS_DEFAULT_REGION=us-east-1 AWS_STACK=73f89cfc-3f99-4227-a339-73a0ba30acbb AWS_APP_ID=1604ff83-aeb4-4677-b436-a9daac1ceb98
      - AWS_ACCESS_KEY_ID=AKIAJSZ63DTL3Z7KLVEQ
      - secure: KRaEGMHtRkYxCmWfvHIEkyfoA/+9EWHcoi1CIoIqXrvsF/ILmVVr0jC7X8u7FdfAiXTqn3jYGtLc5mgo5KXe/8zSLtygCr9U1SKJfwCgsw1INENlJiUraHCQqnnty0b3rsTfoetBnnY0yFIl2g+FUm3A57VnGXH/sTcpDZSqHfjCXivptWrSzE9s4W7+pu4vP+9xLh0sTC9IQNcqQ15L7evM2RPeNNv8dQ+DMdf48915M91rnPkxGjxfebAIbIx1SIhR1ur4rEk2pV4LOHo4ny3sasWyqvA49p1xItnGnpQMWGUAzkr24ggOiy3J5FnL8A9oIkf49RtfK1Z2F0EryA==

  after_success:
    - virtualenv ve && source ve/bin/activate && pip install awscli
    - aws opsworks create-deployment --stack-id $AWS_STACK --app-id $AWS_APP_ID --command '{"Name":"deploy"}'

.. warning::

  Do not change AWS region from ``us-east-1`` even if your instances reside in a different region!
  This is a requirement of OpsWorks at the moment that all the requests are sent to this region, see `the documentation <http://docs.aws.amazon.com/opsworks/latest/userguide/cli-examples.html#cli-examples-create-deployment>`_.

.. note::

  ``AWS_STACK`` and ``AWS_APP_ID`` are not the names of your stack/application, but so called OpsWorks IDs. They can be accessed in stack/application settings page in the OpsWorks Management console.

Connecting to MySQL
^^^^^^^^^^^^^^^^^^^

OpsWorks provides predefined MySQL layer to add to your stack. Connection details for the database are stored in a generated file in the application root.
Type of the file being generated depends on the programming language you defined for your app. For example, for PHP it is ``opsworks.php`` scripts that exposes two classes: ``OpsWorksDb`` and ``OpsWorks``.
You can instantiate these classes to access connection details, as follows:

.. code-block :: php

  require_once("shared/config/opsworks.php");
  $opsWorks = new OpsWorks();
  $db = $opsWorks->db;
  $con = mysqli_connect($db->host, $db->username, $db->password, $db->database);

During tests on Shippable, we need to provide similar file to simulate production environment. For PHP, add the following file to your repository (e.g. under ``test-config/opsworks.php``):

.. code-block :: php

  <?php
  class OpsWorksDb {
    public $adapter, $database, $encoding, $host, $username, $password, $reconnect;

    public function __construct() {
      $this->adapter = 'mysql';
      $this->database = 'test';
      $this->encoding = 'utf8';
      $this->host = '127.0.0.1';
      $this->username = 'shippable';
      $this->password = '';
      $this->reconnect = 'true';
    }
  }

  // ...rest of the file omitted for brevity, you can access it at
  // https://github.com/Shippable/sample-php-mysql-opsworks/blob/master/test-config/opsworks.php

Then, in ``before_script`` step of your build, copy this file to the location required by your application code:

.. code-block :: bash

  before_script: 
    - cp test-config/opsworks.php .

See the full sample of PHP web application featuring MySQL connection `on GitHub <https://github.com/Shippable/sample-php-mysql-opsworks>`_ for details.

General information on using Amazon DynamoDB
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Amazon DynamoDB is a schema-less, fully managed NoSQL database service. It is not a part of OpsWorks offering, but rather a separate service that is accessed using SDK provided
by Amazon.

As DynamoDB is not available for download and is hosted only by Amazon, special care needs to be taken while setting up Shippable build. Connecting to the real DynamoDB from the
integration tests is not an option most of the times, mostly due to cost considerations and time it takes to create a new table in DynamoDB.

For this reason, mock databases were implemented, such as `Dynalite <https://github.com/mhart/dynalite>`_. Comprehensive list of the available mock databases is available on the
`AWS blog <http://aws.amazon.com/blogs/aws/amazon-dynamodb-libraries-mappers-and-mock-implementations-galore/>`_. During our tests it turned out that only the official mock
implementation provided by Amazon (`DynamoDB Local <http://docs.aws.amazon.com/amazondynamodb/latest/developerguide/Tools.DynamoDBLocal.html>`_) worked flawlessly with PHP SDK
and this is the reason why it was included in the samples below. Your mileage may vary, especially as other mock databases catch up with the changes in the SDK.

We also need to inject AWS access key into the production environment, so our application can connect to the DynamoDB API endpoint. There are several ways of realizing this, 
all of which are documented extensively in the `AWS SDK guide <http://docs.aws.amazon.com/aws-sdk-php/guide/latest/credentials.html#credential-profiles>`_.
Here, we will take advantage of the fact that the access key is already available in the Shippable build (in encrypted form, see above) and generate the
configuration file during deployment.

To make the application under test connect to the mock database, we will override ``endpoint`` parameter passed to AWS SDK. Create a JSON file (called ``aws.json`` here)
with following contents:

.. code-block:: json

  {
    "includes": ["_aws"],
    "services": {
      "default_settings": {
        "params": {
          "key": "fake_key",
          "secret": "fake_secret",
          "region": "us-west-2",
          "base_url": "http://localhost:8000"
        }
      }
    }
  }

Supplying ``key``, ``secret`` and a valid region is mandatory, even though they will not be used in the test environment. For this reason, we enter some fake values
to make sure that the application will not be able to reach our production DynamoDB instance.

.. code-block:: bash

  env:
    global:
      - AWS_DEFAULT_REGION=us-east-1 AWS_STACK=73f89cfc-3f99-4227-a339-73a0ba30acbb AWS_APP_ID=1604ff83-aeb4-4677-b436-a9daac1ceb98
      - AWS_ACCESS_KEY_ID=AKIAJSZ63DTL3Z7KLVEQ AWS_REAL_REGION=us-west-2
      - DYNAMODB_LOCAL_DIR=/tmp/dynamodb-local
      - secure: KRaEGMHtRkYxCmWfvHIEkyfoA/+9EWHcoi1CIoIqXrvsF/ILmVVr0jC7X8u7FdfAiXTqn3jYGtLc5mgo5KXe/8zSLtygCr9U1SKJfwCgsw1INENlJiUraHCQqnnty0b3rsTfoetBnnY0yFIl2g+FUm3A57VnGXH/sTcpDZSqHfjCXivptWrSzE9s4W7+pu4vP+9xLh0sTC9IQNcqQ15L7evM2RPeNNv8dQ+DMdf48915M91rnPkxGjxfebAIbIx1SIhR1ur4rEk2pV4LOHo4ny3sasWyqvA49p1xItnGnpQMWGUAzkr24ggOiy3J5FnL8A9oIkf49RtfK1Z2F0EryA==

  before_install:
    - test -e $DYNAMODB_LOCAL_DIR || (mkdir -p $DYNAMODB_LOCAL_DIR && wget http://dynamodb-local.s3-website-us-west-2.amazonaws.com/dynamodb_local_latest -qO- | tar xz -C $DYNAMODB_LOCAL_DIR)

Then, in the ``before_install`` step, we download the latest version of DynamoDB Local and extract it to a temporary location. In ``script`` step we first kill any
outstanding instances of the database, then launch the mock database in the background, saving the process pid in a variable.
We use ``-inMemory`` option here so that the mock database will not save any data to disk. Next, the actual tests are run and we complete the step by shutting down
the database instance.

.. code-block:: bash

  script:
    - ps -ef | grep [D]ynamoDBLocal | awk '{print $2}' | xargs --no-run-if-empty kill
    - java -Djava.library.path=$DYNAMODB_LOCAL_DIR/DynamoDBLocal_lib -jar $DYNAMODB_LOCAL_DIR/DynamoDBLocal.jar -inMemory &
    - DYNAMODB_PID=$!
    # tests run here (language-specific)
    - kill $DYNAMODB_PID

.. note::

  ``grep`` invocation above creates a (somewhat extraneous) character class for the first letter of the search string. This is done to prevent ``grep`` from including itself
  in the results. It works because the ``grep`` process will have ``[D]ynamoDBLocal`` string in its command, which is not matched by ``[D]ynamodblocal`` (because of the square brackets).

Next, we need some way of injecting AWS secret key in the ``aws.json`` file on the target OpsWorks instance. This can be done by registering a Chef deployment hook that will overwrite
this file with values retrieved from Chef configuration. Hooks are registered by placing aptly named files in ``deploy`` directory in your repository root.
Please refer to `AWS documentation <http://docs.aws.amazon.com/opsworks/latest/userguide/workingcookbook-extend-hooks.html>`_
and `Opscode documentation on deploy resource <http://docs.opscode.com/resource_deploy.html#deploy-phases>`_ if you interested in details.

For your convenience, here (and in samples repositories) we provide a ``before_restart`` hook that will generate correct ``aws.json``.
Please note that we don't define ``endpoint`` here, so AWS will pick the correct one based on the region.
Place this file as ``deploy/before_restart.rb`` in your repository root:

.. code-block:: ruby

  require 'json'

  Chef::Log.info('Generating aws.json configuration file')

  aws_config = {
    :includes => ['_aws'],
    :services => {
      :default_settings => {
        :params => {
          :key => node[:dynamodb][:aws_key],
          :secret => node[:dynamodb][:aws_secret],
          :region => node[:dynamodb][:region]
        }
      }
    }
  }

  aws_file_path = ::File.join(release_path, 'aws.json')
  file aws_file_path do
    content aws_config.to_json
    owner new_resource.user
    group new_resource.group
    mode 00440
  end

The script above reads the required configuration variables from the Chef node attributes and saves them as JSON file in the format expected by AWS SDKs.

While launching deployment, we can override node attributes by passing `custom JSON <http://docs.aws.amazon.com/opsworks/latest/userguide/workingstacks-json.html>`_. We will take
advantage of this option to set node attributes that the hook above expects.
The special syntax with ``>`` sign is used here to prevent YAML parser from interpreting colons in the JSON definition.

.. code-block:: bash

  after_success:
    - >
      DEPLOY_JSON=$(printf '{"dynamodb": {"aws_key": "%s", "aws_secret": "%s", "region": "%s"}}' $AWS_ACCESS_KEY_ID $AWS_SECRET_ACCESS_KEY $AWS_REAL_REGION)
    - virtualenv ve && source ve/bin/activate && pip install awscli
    - aws opsworks create-deployment --stack-id $AWS_STACK --app-id $AWS_APP_ID --command '{"Name":"deploy"}' --custom-json "$DEPLOY_JSON"

Then proceed to configure your application as is outlined in per-language guides below.

Using DynamoDB with PHP
^^^^^^^^^^^^^^^^^^^^^^^

To access DynamoDB, you need some client library that is able to speak AWS API. We will use the official `AWS PHP SDK <http://aws.amazon.com/sdkforphp/>`_ in the sample below.
We will install the library using `Composer <https://getcomposer.org/>`_. Create ``composer.json`` in the root of your repository with the following contents:

.. code-block:: json

  {
    "require": {
      "aws/aws-sdk-php": "2.*"
    }
  }

Composer will be already available on Shippable minion. Install the dependencies during ``before_script`` step as follows:

.. code-block:: bash

  before_script: 
    - mkdir -p shippable/testresults
    - mkdir -p shippable/codecoverage
    - composer install

Then, we need to perform the same step on the target OpsWorks instance. Add the following deploy hook as ``deploy/before_symlink.rb``:

.. code-block:: ruby

  run "cd #{release_path} && ([ -f tmp/composer.phar ] || curl -sS https://getcomposer.org/installer | php -- --install-dir=tmp)"
  run "cd #{release_path} && php tmp/composer.phar --no-dev install"

We can then proceed to consume ``aws.json`` file we created in the previous section to instantiate AWS SDK client:

.. code-block:: php

  require('vendor/autoload.php');
  use Aws\Common\Aws;

  $aws = Aws::factory('aws.json');
  $client = $aws->get('DynamoDb');

This client can be then used to interact with DynamoDB, for example as follows:

.. code-block:: php

  $client->createTable(array(
    'TableName' => self::TABLE_NAME,
    'AttributeDefinitions' => array(
      array(
        'AttributeName' => 'id',
        'AttributeType' => 'N'
      )
    ),
    'KeySchema' => array(
      array(
        'AttributeName' => 'id',
        'KeyType' => 'HASH'
      )
    ),
    'ProvisionedThroughput' => array(
      'ReadCapacityUnits' => 1,
      'WriteCapacityUnits' => 1
    )
  ));

Refer to the `DynamoDB client documentation <http://docs.aws.amazon.com/aws-sdk-php/guide/latest/service-dynamodb.html>`_
and `the full sample <https://github.com/Shippable/sample-php-dynamo-opsworks>`_ on our GitHub account for details.

Using DynamoDB with Node.js
^^^^^^^^^^^^^^^^^^^^^^^^^^^

To access DynamoDB, you need some client library that is able to speak AWS API. We will use the official `AWS Node.js SDK <http://aws.amazon.com/sdkfornodejs/>`_ in the sample below.
We will install the library using ``npm`` (saving the dependency to ``package.json``):

.. code-block:: bash

  npm install --save aws-sdk

The packages will be then installed automatically (by invoking ``npm install``) both by Shippable and OpsWorks deployment recipe.

Configuration file (that we called ``aws.json``) has slightly different structure for Node.js SDK. For Shippable build environment it
will look as follows:

.. code-block:: json

  {
    "accessKeyId": "fake_key",
    "secretAccessKey": "fake_secret",
    "region": "us-west-2",
    "endpoint": "http://localhost:8000"
  }

We also need to slightly change the Chef deployment hook for the modified JSON structure:

.. code-block:: ruby

  require 'json'

  return unless node[:dynamodb]
  Chef::Log.info('Generating aws.json configuration file')

  aws_config = {
    :accessKeyId => node[:dynamodb][:aws_key],
    :secretAccessKey => node[:dynamodb][:aws_secret],
    :region => node[:dynamodb][:region]
  }

  aws_file_path = ::File.join(release_path, 'aws.json')
  file aws_file_path do
    content aws_config.to_json
    owner new_resource.user
    group new_resource.group
    mode 00440
  end

DynamoDB client can be then constructed with the following snippet:

.. code-block:: javascript

  var AWS = require('aws-sdk');
  AWS.config.loadFromPath('./aws.json');
  var db = new AWS.DynamoDB();

Next, the client can be used to interact with DynamoDB, for example as follows:

.. code-block:: javascript

  var params = {
    TableName: TABLE_NAME,
    Item: {
      id: {
        N: '1'
      },
      score: {
        N: String(score)
      }
    }
  };
  db.putItem(params, callback);

Refer to the `DynamoDB client documentation <http://docs.aws.amazon.com/AWSJavaScriptSDK/latest/AWS/DynamoDB.html>`_
and `the full sample <https://github.com/Shippable/sample-nodejs-dynamo-opsworks>`_ on our GitHub account for details.

Continuous deployment to Google App Engine
..........................................

Google App Engine supports Python, PHP, Go and Java applications. Support for PHP is in preview, while for Go is marked as experimental. As runtime present on App Engine is a very specific one, with many Google-specific services and some blacklisted modules, it is recommended to use Google App Engine SDK both during the development and testing of your application.

Installation of the GAE SDK
^^^^^^^^^^^^^^^^^^^^^^^^^^^

SDKs for all the runtimes are available as ZIP downloads on `the Google App Engine page <https://developers.google.com/appengine/downloads>`_.
The SDK contains tools to interact with the GAE API: for instance, it allows deployment of the application. 
Moreover, it comes with Development Server that lets you test the application on your local machine (and on Shippable minion), simulating
the GAE environment. Stubs for the GAE services are also provided to make unit testing easier.

Download the SDK for your platform from the link above to begin working on the application.
To make the SDK available for your Shippable build (here, for a Python project), add the following ``before_install`` step:

.. code-block:: bash

  env:
    global:
      - GAE_DIR=/tmp/gae

  before_install:
    - >
      test -e $GAE_DIR || 
      (mkdir -p $GAE_DIR && 
       wget https://storage.googleapis.com/appengine-sdks/featured/google_appengine_1.9.6.zip -q -O /tmp/gae.zip &&
       unzip /tmp/gae.zip -d $GAE_DIR)

It will first test if the tools are already available and download & unzip them if there are not.

Using Datastore from Python
^^^^^^^^^^^^^^^^^^^^^^^^^^^

Google App Engine offers a number of storage services. One of them is NDB Datastore that is instantly available to your application, once
you deploy it to the platform.
To interact with Datastore, you need to use libraries bundled with the SDK. Below is a simple example of code that stores and retrieves data from 
Datastore. More information can be found in `the GAE documentation <https://developers.google.com/appengine/docs/python/ndb/>`_:

.. code-block:: python

  from google.appengine.ext import ndb

  class Score(ndb.Model):
    score = ndb.IntegerProperty()
    timestamp = ndb.DateTimeProperty(auto_now_add=True)

  class Storage():
    def score_key(self):
      return ndb.Key('Score', 'Store')

    def populate(self):
      new_score = Score(parent=self.score_key())
      new_score.score = random.randint(1, 1234)
      new_score.put()

    def get_score(self):
      score_query = Score.query(ancestor=self.score_key()).order(-Score.timestamp)
      return score_query.get().score

No connection setup is required, as the GAE will handle providing the service to your application automatically.
Thanks to the existence of the Development Server, we can test this code both with a unit test and a integration test.

Unit test that stubs Datastore calls looks as follows:

.. code-block:: python

  import unittest
  from google.appengine.ext import db
  from google.appengine.ext import testbed
  from helloworld import Storage

  class HelloTestCase(unittest.TestCase):
    def setUp(self):
      self.testbed = testbed.Testbed()
      self.testbed.activate()
      self.testbed.init_datastore_v3_stub()

    def tearDown(self):
      self.testbed.deactivate()

    def test(self):
      storage = Storage()
      storage.populate()
      score = storage.get_score()
      self.assertLess(score, 1234)

  if __name__ == "__main__":
    unittest.main()

The only GAE-specific code is enclosed in ``setUp`` and ``tearDown`` methods and it initializes and then closes the stubbing framework.

We can also write an integration test, in which the code connects to a mock database included in the SDK:

.. code-block:: python

  from webtest import TestApp
  from helloworld import application

  app = TestApp(application)

  def test_index():
    response = app.get('/')
    assert 'Hello, World' in response

For the above to work, we need to use a dedicated test runner that will run the test in the Development Server environment.
For example, to use `NoseGAE <https://github.com/Trii/NoseGAE>`_,  we need to install the following modules (preferably listed in ``requirements.txt``
file):

.. code-block:: bash

  nose
  coverage
  NoseGAE
  WebTest

We then install them on Shippable minion, using the following ``install`` step:

.. code-block:: bash

  install:
    - pip install -r requirements.txt

Finally, we can launch the both tests by invoking the test runner with extra arguments during the ``script`` step:

.. code-block:: bash

  script:
    - >
      nosetests test.py func_test.py 
      --with-gae --without-sandbox --gae-lib-root=$GAE_DIR/google_appengine
      --with-xunit --xunit-file=shippable/testresults/test.xml
      --with-coverage --cover-xml --cover-xml-file=shippable/codecoverage/coverage.xml

Please note the second line of the command, where we turn on the GAE plugin and pass the path of the SDK installation on the minion.
The ``--without-sandbox`` option was required to have the tests working successfully. This NoseGAE option tries to simulate the GAE environment,
where some functions are prohibited. Apparently, it doesn't work correctly for Datastore services.

The other parameters are here to generate JUnit XML report in the location expected by Shippable, as well as the coverage report.

.. _gae_python_deployment:

Deployment of a Python application
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

After you create the application using the GAE Admin Console, you can deploy it using the ``appcfg`` tool from the SDK.
First, create ``app.yaml`` file, including your application name in the first line:

.. code-block:: bash

  application: sample-python-datastore
  version: 1
  runtime: python27
  api_version: 1
  threadsafe: 1

  handlers:
  - url: /.*
    script: helloworld.application

Then, we need to authenticate against the GAE API. We have two options here that are suitable for non-interactive build environment:
password-based authentication and OAuth2.

To setup password-based authentication, include two environment variables in your ``shippable.yml``:
``EMAIL`` (that stores the name of your Google account) and ``GAE_PASSWORD``.
It is recommended to store the password in encrypted form, using :ref:`secure_env_variables`:

.. code-block:: bash

  env:
    global:
      - GAE_DIR=/tmp/gae
      - EMAIL=shippable@gmail.com
      - secure: lffPR8giDdKinq1LfjTabgM8Lufb3sdweFWJcoU8o/KIvwTg9NOxEw3oG5pw4+pI0c3q/k0JkBv7QgDGkoiRHwZkebWYNcHwyo2NFaa/cpwpNjv3pMZsXpMiw+duSvfjA/XmFAynmW8/ft2YaAzpB1Mbn5p2k7ID2qCMv/YmFgIu605VK/WUnYPEdxMD2vkifVSNAIH42GOR+2ht4nKj85Wsu9OGgMBJ5XAqVcQoWX+Ui9yZvtaf3WKzowg+MC4PQ0qGLH/l6WHkY8bBCduMz65JjZIss2s972L4P8Hwpk+gDdVtRE82hKH7GuEYdNKhKjbthZmn5AF4thI72N5TjQ==

Next, you can invoke ``appcfg update`` command to deploy new version of your application with the following ``after_success`` step:

.. code-block:: bash

  after_success:
    - echo "$GAE_PASSWORD" | $GAE_DIR/google_appengine/appcfg.py -e "$EMAIL" --passin update .

.. note::

  If you use two-factor authentication for your Google account, you need to generate application-specific password for the GAE to use.
  Refer to `this documentation <https://support.google.com/accounts/answer/185833?hl=en>`_ for the details.

Alternatively, you can use OAuth2 protocol to authenticate against the GAE API. To set it up, first run this command in the repository root on your
local workstation:

.. code-block:: bash

  $PATH_TO_GAE_SDK/appcfg.py --oauth2 list_versions .

It will open a page in your browser where you can authorize the GAE to access your Google account.
As the result, ``.appcfg_oauth2_tokens`` file will be created in your home directory, containing the access token.
You can then encrypt it as Shippable secure variable and use in your ``after_success`` step as follows:

.. code-block:: bash

  after_success:
    - $GAE_DIR/google_appengine/appcfg.py --oauth2_access_token=$GAE_TOKEN update .

.. note::

  Recently, Google opened a preview of git-based deployment workflow, in which you push the code to a git repository, triggering the build.
  As this functionality is not yet in its final form, it is not discussed here. Please refer to
  `the GAE documentation <https://developers.google.com/cloud/devtools/repo/push-to-deploy>`_ to track its progress.

Full sample of Python+Datastore application can be found on `our Github account <https://github.com/Shippable/sample-python-datastore-appengine>`_.

Using Datastore from Go
^^^^^^^^^^^^^^^^^^^^^^^

To interact with Datastore from Go, you need to use libraries bundled with the SDK. Below is a simple example of code that stores and retrieves data from 
Datastore. More information can be found in `the GAE documentation <https://developers.google.com/appengine/docs/go/datastore/>`_:

.. code-block:: go

  import (
    "math/rand"
    "time"

    "appengine"
    "appengine/datastore"
  )

  type Score struct {
    Score int
    Date  time.Time
  }

  func scoreKey(c appengine.Context) *datastore.Key {
    return datastore.NewKey(c, "Scores", "default_scoreboard", 0, nil)
  }

  func populate(c appengine.Context) error {
    score := Score{
      Score: rand.Intn(1234),
      Date:  time.Now(),
    }
    key := datastore.NewIncompleteKey(c, "Score", scoreKey(c))
    _, err := datastore.Put(c, key, &score)
    return err
  }

  func getScore(c appengine.Context) (int, error) {
    query := datastore.NewQuery("Score").Ancestor(scoreKey(c)).Order("-Date").Limit(1)
    for t := query.Run(c); ; {
      var score Score
      if _, err := t.Next(&score); err != nil {
        return -1, err
      }

      return score.Score, nil
    }
  }

No connection setup is required, as the GAE will handle providing the service to your application automatically.

Unit test that stubs Datastore calls using `aetest package <https://godoc.org/code.google.com/p/appengine-go/appengine/aetest>`_ looks as follows:

.. code-block:: go

  import (
    "testing"

    "appengine/aetest"
  )

  func TestStorage(t *testing.T) {
    c, err := aetest.NewContext(nil)
    if err != nil {
      t.Fatal(err)
    }
    defer c.Close()

    if err := populate(c); err != nil {
      t.Fatal(err)
    }

    score, err := getScore(c)
    if err != nil {
      t.Fatal(err)
    }
    if score < 0 || score > 1023 {
      t.Errorf("Score outside of expected range: %d", score)
    }
  }

.. note::

  Full integration testing of GAE Go applications with automatic mocking of the services is not yet available,
  but work on it is `being performed by Google team <https://groups.google.com/d/msg/google-appengine-go/9JZDLUMRkRE/B_UOS44UQjkJ>`_.

For the above to work, we need to run the tests via ``goapp`` command that is supplied as part of the GAE Go SDK.
Its installation and setup is described in the section below.

Deployment of a Go application
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Go packages are resolved relative to ``GOPATH`` variable that needs to be set both in your development environment and on Shippable minion.
`The common practice <http://code.google.com/p/go-wiki/wiki/GithubCodeLayout>`_ when structuring an application that is hosted on GitHub
is to name your packages according to the following pattern:

.. code-block:: bash

  github.com/<GitHub username>/<repository name>/<package name, probably nested>

For example, the package that houses the main HTTP handler in `our Go sample <https://github.com/Shippable/sample-go-datastore-appengine>`_
is called ``github.com/Shippable/sample-go-datastore-appengine/hello``. It follows that in your development environment the contents of
the sample repository would be stored in the ``$GOPATH/src/github.com/Shippable/sample-go-datastore-appengine`` path.

Adhering to this convention ensures that the testing tools (which are package-aware) will work correctly and that your package can be
consumed by other packages.

Google App Engine slightly diverges from this structure, expecting to find the main entry point for the application in the root of your
repository.
In other words, while ``goapp test`` command lives in a package-oriented world, ``goapp serve`` and ``goapp deploy`` are tied to the
current directory.
Hence, it is common to create a dispatcher in the root of the repository that then calls the individual packages:

.. code-block:: go

  package routes

  import (
    "net/http"

    "github.com/Shippable/sample-go-datastore-appengine/hello"
  )

  func init() {
    http.HandleFunc("/", hello.Handler)
  }

This way, we can test the individual packages, while ensuring that the application will deploy properly.
The following snippet from ``shippable.yml`` downloads the GAE Go SDK, installs the packages required for generation of test and
coverage reports and then links the repository to the correct place in Go workspace (root of which is identified by ``GOPATH``).
Of note are also the environment variables used to authenticate against the GAE API.
Please refer to :ref:`gae_python_deployment` for details on different methods of authentication.

.. code-block:: yaml

  env:
    global:
      - GAE_DIR=/tmp/go_appengine
      - EMAIL=shippable@gmail.com
      - secure: lffPR8giDdKinq1LfjTabgM8Lufb3sdweFWJcoU8o/KIvwTg9NOxEw3oG5pw4+pI0c3q/k0JkBv7QgDGkoiRHwZkebWYNcHwyo2NFaa/cpwpNjv3pMZsXpMiw+duSvfjA/XmFAynmW8/ft2YaAzpB1Mbn5p2k7ID2qCMv/YmFgIu605VK/WUnYPEdxMD2vkifVSNAIH42GOR+2ht4nKj85Wsu9OGgMBJ5XAqVcQoWX+Ui9yZvtaf3WKzowg+MC4PQ0qGLH/l6WHkY8bBCduMz65JjZIss2s972L4P8Hwpk+gDdVtRE82hKH7GuEYdNKhKjbthZmn5AF4thI72N5TjQ==

  before_install:
    - >
      test -e $GAE_DIR || 
      (mkdir -p $GAE_DIR && 
       wget https://storage.googleapis.com/appengine-sdks/featured/go_appengine_sdk_linux_amd64-1.9.6.zip -q -O /tmp/gae.zip &&
       unzip /tmp/gae.zip -d /tmp)
    - go get github.com/jstemmer/go-junit-report
    - go get github.com/t-yuki/gocover-cobertura
    - mkdir -p $GOPATH/src/github.com/Shippable
    - ln -sfn $PWD $GOPATH/src/github.com/Shippable/sample-go-datastore-appengine

Finally, we can launch the test by invoking the test runner with extra arguments during the ``script`` step:

.. code-block:: yaml

  script:
    - >
      $GAE_DIR/goapp test -v -coverprofile=shippable/codecoverage/coverage.out github.com/Shippable/sample-go-datastore-appengine/hello |
        $GOPATH/bin/go-junit-report > shippable/testresults/results.xml
    - $GOPATH/bin/gocover-cobertura < shippable/codecoverage/coverage.out > shippable/codecoverage/coverage.xml

Please note that we use ``goapp`` command from the GAE SDK instead of the standard ``go`` command.
This is required in order to be able to use ``aetest`` package.

Next, we need to create the application using the GAE console and create ``app.yml`` file with matching application name:

.. code-block:: yaml

  application: sample-go-datastore
  version: 1
  runtime: go
  api_version: go1

  handlers:
  - url: /.*
    script: _go_app

Finally, we can deploy the application to Google App Engine in ``after_success`` step:

.. code-block:: yaml

  after_success:
    - echo "$GAE_PASSWORD" | $GAE_DIR/appcfg.py -e "$EMAIL" --passin update .

----------

**Pull Request**
----------------


Shippable will integrate with github to show your pull request status on CI. Whenever a pull request is opened for your repo, we will run the build for the respective pull request and notify you about the status. You can decide whether to merge the request or not, based on the status shown. If you accept the pull request, Shippable will run one more build for the merged repo and will send email notifications for the merged repo.

 
--------

**Collaborators**
------------------

Shippable will automatically add your github collaborators when you create a project and by default they will be assigned the role of **Build engineer**. You can see the list of collaborators or change their role by expanding your repo on the settings page.


There are two types of roles that users can have -

**Owner :** 
Owner is the highest role. This role permits users to create, run and delete a project. 


**Collaborators :** 
Collaborators can run or manage projects that are already setup. They have full visibility into the project and can trigger the build.


--------

**Build Termination**
-----------------------


If your script or test suite hangs for a long time or there hasn't been any log output in 20 minutes, then Shippable will forcefully terminate the build and add a message to the console log.

--------

**Skipping a build**
-----------------------

Any changes to your source code will trigger a build automatically on Shippable. So if you do not want to run build for a particular commit, then add **[ci skip]** or **[skip ci]** to your commit message. 

Our webhook processor will look for the string  **[ci skip]** or **[skip ci]** in the commit message and if it exists, we do not create build for that commit.
