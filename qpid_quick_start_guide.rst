Qpid C++ broker Quick Start Guide
=================================


General
-------

If you’re new to Qpid, this guide offers useful tips to help you find your way
to setup Qpid C++ broker in very basic configuration and how to use Qpid tools.

1. Introduction
2. Installation
3. Configuration
4. Running qpid
5. Running qpid in HA
6. Testing and troubleshooting


1. Introduction
---------------

Apache Qpid™ C++ broker is a cross-platform AMQP messaging daemon which supports
many languages and platforms. This document is a step-by-step guide which
describes basic installation and configuration of Qpid C++ broker.

**Basic assumption**

 * Qpid config file: ``/etc/qpidd.conf:``.
 * Qpid daemon runs as a ``qpidd:qpidd`` user:group.
 * Qpid version 0.22 or higher.


2.Installation
--------------

Most of modern Linux distributions provide ready to install Qpid binary packages
but some configuration details may differ depends on the dist flavour.

 * **CentOS 6.5.** CentOS 7 does not provide Qpid packages.

   ``# yum install -y qpid-cpp-server qpid-cpp-server-ssl qpid-cpp-client
   qpid-tools``

   **(!!!)** SSL support does not come with base packages,
   ``qpid-cpp-server-ssl`` is required.
   CentOS 7 does not provide Qpid packages at all.

 * **Debian 7.5 Wheezy**

   ``# aptitude install qpidd qpid-client qpid-tools qpid-doc``


3. Configuration
----------------

There is a wide range of possible Qpid configurations below examples presents

 a. basic configuration
 b. Qpid AMQP over SSL
 c. Qpid in HA pair configuration

Qpidd daemon can be started without any configuration provided, in this way all
default values will be taken. This is not recomended way of running Qpid but can
be used for troubleshooting purposes.

To list all possible configuration options execute below command::

 /usr/sbin/qpidd --help

All the options have a short description and the default values are shown in the brackets.
Every option listed in ``qpidd --help`` output can be use in configuration file.

Below examples are not ment to be used for production environments, neither
security nor access control aspects are included in them.

 **3a. Basic configuration.**

  This configuration runs Qpid broker on default unencrypted AMQP port 5672, with
  no user Authentication, with no local data store and logging to syslog.
  This configuration allows to run Qpid broker as long as port 5672 is
  fee to use. Configuration file example::

   auth=no
   no-data-dir=yes
   log-enable=info+
   log-to-syslog=yes
   port=5672

 **3b. Running Qpid over SSL.** 

  SSL transport is a very common requirement. In most of the cases self signed
  SSL certificate is used despite the security weaknes it brings.
  Qpid requers Mozilla's Network Security Services Library (NSS) for SSL support
  and is managed by certutil extrnal utility, not provided by Qpid.

 **3b1. Qpid certificate and SSL store location.**

  In most of Lunux distros, PKI implementation keeps SSL certificates in
  */etc/pki* directory. It is possible to use system wide PKI or create separate
  store just for Qpid usage and the second options is prefered for simple
  deployments.

  * Set and export below variables. This help us to avoid mistakes and allows
    copy/paste below commands::

     export CERT_DIR='/etc/qpid'
     export CERT_PW_FILE="${CERT_DIR}/pwfile"
     export NICKNAME='qpid_broker'

  * Create new dedicated for Qpid PKI store with pseudorandom password::

     test -d ${CERT_DIR} || mkdir ${CERT_DIR}
     echo "$(date) $(uptime) $(uname -a)" | md5sum | cut -d' ' -f1 > ${CERT_PW_FILE}
     certutil -N -d ${CERT_DIR} -f ${CERT_PW_FILE}

    **(!!!)** Before re-run above commands make sure *${CERT_PW_FILE}*
    direcory is empty.

  * At this point, there is self signed CA in ``${CERT_DIR}``::

     certutil -d ${CERT_DIR} -L

  * Generate Qpid certificate. The NICKNAME is required and it will be used in
    Qpid configuration file later.::

     certutil -S -d ${CERT_DIR} -n ${NICKNAME} -s "CN=${NICKNAME}" -t "CT,," -x -f ${CERT_PW_FILE} -z /usr/bin/certutil

  * Give read access right to qpidd group::

     chmod -Rv g+r ${CERT_DIR}
     chgrp -Rv qpidd ${CERT_DIR}
     chmod -Rv o-rwx ${CERT_DIR}

  * List all certificates in new store::
    
     certutil -d ${CERT_DIR} -L

  * Prepare Qpid config file **/etc/qpidd.conf**::

     echo -e "\
     auth=no\n\
     no-data-dir=yes\n\
     log-enable=info+\n\
     log-to-syslog=yes\n\
     port=5672\n\
     ssl-port=5671\n\
     ssl-cert-password-file=${CERT_PW_FILE}\n\
     ssl-cert-db=${CERT_DIR}\n\
     ssl-cert-name=${NICKNAME}\n\
     ssl-require-client-authentication=no" \
         > /etc/qpidd.conf

  * Restart Qpid daemon using::

     /etc/init.d/qpidd start
     service qpidd start
     systemctl start qpidd

  * Verify Qpid daemon is accessible on 5671::

     ss -ltp
     netstat -nlp -t
     openssl s_client -connect localhost:5671

    Check if you see below line in the log file::

     [Security] notice Listening for SSL connections on TCP/TCP6 port 5671

 **3c. Qpid high avaiability (HA) configuration.**

  To configure Qpid in HA, follow point **3a** or **3b** to setup two qpid
  instances on two separate hosts, in below example hostnames are: qpid1,
  qpid2. The configuration files should be identical, example::

   auth=no
   no-data-dir=yes
   log-enable=info+
   log-to-syslog=yes
   port=5672
   ha-cluster=yes
   ha-brokers-url=amqp:tcp:qpid1:5672,tcp:qpid2:5672
   ha-replicate=all
   ha-username=ha_qpid
   ha-password=q_ha_pass
   ha-mechanism=PLAIN

  Having two qpid nodes up and running in this basic configuration here is how
  to setup Qpid active/backup cluster.

  * Cheking initial status of both nodes::

     [root@qpid1 ~]# qpid-ha status --all
     qpid1:5672 joining
     qpid2:5672 joining

  * Promoting one of the nodes to being a master node of the cluster
    and checking status again::

     [root@qpid1 ~]# qpid-ha promote
     [root@qpid1 ~]# qpid-ha status --all
     qpid1:5672 active
     qpid2:5672 joining

  * We need to give clusert a little bit time to form the cluster.

    ::

     [root@qpid1 ~]# qpid-ha status --all
     qpid1:5672 active
     qpid2:5672 ready

  * Cluster status with one of the hosts down::

     [root@qpid1 ~]# qpid-ha  status --all
     qpid1:5672 active
     qpid2:5672 [Errno 111] Connection refused


6. Testing and troubleshooting
------------------------------

 **a. Run qpidd in foreground**

  ::

   qpidd --config /etc/qpidd.conf

 **b. Enable extra logging**

  Qpid allows to configure logging subsystem in very sophisticated manner
  but for sake of troubleshooting below example shows how to run qpidd in
  foreground with debug logging enabled::

   /usr/sbin/qpidd --config /etc/qpidd.conf --log-enable debug+

 **c. Missing Qpid modules/plugins**

  Qpid modules and extensions are located in: ``/usr/lib64/qpid/daemon/``.
  Qpidd reporting unknown configuration options may be a sign of missing module.

  Example, missing HA module ``ha.so``::

   2014-08-20 18:34:12 [Broker] critical Unexpected error: Error in configuration file /etc/qpidd.conf: Bad argument: |ha-cluster=yes|

  To check which modules are loaded you can execute below command and search for
  shared libraries loaded from ``/usr/lib64/qpid/daemon/``.

  ::

   lsof -n -p $(pgrep qpidd)


 **d. Problems with reading SSL certificates or keys**

  These are very common problems and many times below errors mislead and make
  problem more complicated than it is.

  Errors::

   Jun 22 11:10:43 qpid1 qpidd[739]: 2014-06-22 11:10:43 error Failed to initialise SSL plugin: Failed: NSS error [-8015] (qpid/sys/ssl/util.cpp:103)
   certutil: function failed: SEC_ERROR_LEGACY_DATABASE: The certificate/key database is in an old, unsupported format.
   [root@os-mysql1 ha_qpid]# certutil -L -d /etc/pki/qpidd/
   certutil: function failed: SEC_ERROR_LEGACY_DATABASE: The certificate/key database is in an old, unsupported format.
   [root@os-mysql1 ha_qpid]# sudo -u qpidd /usr/sbin/qpidd --config /etc/qpidd.conf
   certutil: function failed: SEC_ERROR_LEGACY_DATABASE: The certificate/key database is in an old, unsupported format.

  All above errors are caused by incorrect permissions on SSL certificate store.
  Qpid daemon runs as unprivileged user which does not have read access to SSL
  certificate and private key.

 **e. Qpid node unable connect to master node**

  Error exampl:

  ::

   [root@qpid2 /]# qpid-ha  status  --all
   qpid1:5672 [Errno 113] No route to host
   qpid2:5672 joining
   [root@qpid2 /]# telnet qpid1 5672
   Trying 192.168.94.103...
   telnet: connect to address 192.168.94.103: No route to host

  Problem may be caused by closed by firewall port on the master node.


 **f. Checking Qpid status**

  Qpid comes with set of tools, one of which is ``qpid-stat``. It allows examine
  varius qpid statistics.

  ::

   [root@h102 radek]# qpid-stat  -e
   Exchanges
   exchange            type     dur  bind  msgIn  msgOut  msgDrop  byteIn  byteOut  byteDrop
   ===========================================================================================
   qmf.default.direct  direct           1    69     69       0     76.3k   76.3k       0
   amq.direct          direct   Y       1   522    522       0      212k    212k       0
   amq.topic           topic    Y       0     0      0       0        0       0        0
   qpid.management     topic            3   470     78     392      181k   35.0k     146k
   amq.fanout          fanout   Y       0     0      0       0        0       0        0
   amq.match           headers  Y       0     0      0       0        0       0        0
   qmf.default.topic   topic            1   479     89     390      518k    109k     409k

  If Qpid requiers authentication ``qpid-stat`` command should looke like this:

  ::

   [root@h102 radek]# qpid-stat  -c admin/1qazs@localhost:5672
   Connections
   client-addr                     cproc      cpid  auth        connected  idle  msgIn  msgOut
   =============================================================================================
   127.0.0.1:5672-127.0.0.1:39928  qpid-stat  3969  admin@QPID  2s         0s     251    320
