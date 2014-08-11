Qpid Quick Start Guide

If you’re new to Qpid, this guide offers useful tips to help you find your way
to setup this service.

1 Introduction
2 Installation
3 Initial Configuration
4 Running qpid
5 Running qpid in HA
5 Testing and troubleshooting
7 Additional notes

1 Introduction
Apache Qpid™ is a cross-platform AMQP messaging broker that supports many
languages and platforms. This document is a step-by-step guide which describes
basic installation and configuration of Qpid broker.


2.Installation
Most of modern Linux distributions provide ready to install Qpid binary packages
but some configuration details may differ depends on the dist flavour.

2.1 CentOS 6.5
# yum install -y qpid-cpp-server qpid-cpp-server-ssl qpid-cpp-client qpid-tools
!!! SSL support does not come with base packages, qpid-cpp-server-ssl is required.

2.1 Debian 7.5 wheezy
# aptitude install qpidd qpid-client qpid-tools qpid-doc


3. Configuration
There is a wide range of possible Qpid configurations below examples presents
a. basic configuration
b. AMQP over SSL with Qpid
c. Qpid in HA configuration
In fact qpidd can be started without any configuration provided, in this way all
default values will be taken. This is not recomended way of running Qpid but can
be used for troubleshooting purposes.

Below examples are not ment to be used for production environments, neither
security nor access control aspects are included in them.

3a. Basic configuration.
This configuration runs Qpid broker on default unencrypted AMQP port 5672, with
no user Authentication, with no local data store and logging to syslog.
This configuration allows to run Qpid broker as long as port 5672 is fee to use.

/etc/qpidd.conf:
auth=no
no-data-dir=yes
log-enable=info+
log-to-syslog=yes
port=5672

3b. SSL transport is a very common requirement. In most of the cases self signed
SSL certificate is used despite the security weaknes it brings.

Qpid requers Mozilla's Network Security Services Library (NSS) for SSL support
and is managed by certutil extrnal utility, not provided by Qpid.

3b1. Qpid certificate and SSL store location.
In most of Lunux distros, PKI implementation keeps SSL certificates in /etc/pki
directory. It is possible to use system wide PKI or create separate store just
for Qpid usage and the second options is prefered for simple deployments.
- Create new dedicated for Qpid PKI store
   export CERT_DIR='/etc/qpid'
   export CERT_PW_FILE="${CERT_DIR}/pwfile"
   mkdir ${CERT_DIR}
   echo "$(date) $(uptime) $(uname -a)" | md5sum | cut -d' ' -f1 > ${CERT_PW_FILE}
   certutil -N -d ${CERT_DIR} -f ${CERT_PW_FILE}

At this point, there is self signed CA in ${CERT_DIR}
   certutil -d ${CERT_DIR} -L

- Generate Qpid certificate. The NICKNAME is required and it will be used in
  Qpid configuration file later.
   export NICKNAME="qpid_broker"
   certutil -S -d ${CERT_DIR} -n ${NICKNAME} -s "CN=${NICKNAME}" -t "CT,," -x -f ${CERT_PW_FILE} -z /usr/bin/certutil

- Grant read access right to qpidd group
   chmod -Rv g+r  qpidd /etc/qpid
   chgrp -Rv qpidd /etc/qpid
   chmod -Rv o-rwx /etc/qpid

List all certificates in new store:
   certutil -d ${CERT_DIR} -L

/etc/qpidd.conf:
auth=no
no-data-dir=yes
log-enable=info+
log-to-syslog=yes
port=5672
ssl-port =5671
ssl-cert-password-file=${CERT_PW_FILE}
ssl-cert-db=${CERT_DIR}
ssl-cert-name=${NICKNAME}
ssl-require-client-authentication=no

/etc/init.d/qpidd start
service  qpidd start
systemctl start qpidd

ss -ltp


3c. Qpid high avaiability (HA) configuration.
To configure Qpid in HA, follow point 3a or 3b to setup two qpid instances
on two separate hosts, in below example hostnames: qpid1, qpid2.

/etc/qpidd.conf:


4. Checking status of working Qpid.



root@d64:~# qpid-stat -c admin/1qazs@localhost:5672
Connections
  client-addr                     cproc      cpid  auth        connected  idle  msgIn  msgOut
  =============================================================================================
  127.0.0.1:5672-127.0.0.1:39928  qpid-stat  3969  admin@QPID  2s         0s     251    320


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
                      direct           5     0      0       0        0       0        0

root@d64:~# qpid-stat -c admin/1qazs@localhost:5672
Connections
  client-addr                     cproc      cpid  auth        connected  idle  msgIn  msgOut
  =============================================================================================
  127.0.0.1:5672-127.0.0.1:39915  qpid-stat  3735  admin@QPID  2s         0s     251    320


            qpid-printevents --sasl-mechanism PLAIN   admin/1qazs@localhost:5672
root@d64:~# qpid-printevents  admin/1qazs@localhost:5672
Tue Jun 17 22:54:26 2014 NOTIC qpid-printevents:brokerConnected broker=localhost:5672
Tue Jun 17 22:54:28 2014 INFO  org.apache.qpid.broker:bind broker=localhost:5672 rhost=127.0.0.1:5672-127.0.0.1:39918 user=admin@QPID exName=qpid.management qName=topic-d64.3775.1 key=console.event.# args={}
Tue Jun 17 22:54:28 2014 INFO  org.apache.qpid.broker:bind broker=localhost:5672 rhost=127.0.0.1:5672-127.0.0.1:39918 user=admin@QPID exName=qmf.default.topic qName=qmfc-v2-hb-d64.3775.1 key=agent.ind.heartbeat.# args={}
Tue Jun 17 22:54:28 2014 INFO  org.apache.qpid.broker:bind broker=localhost:5672 rhost=127.0.0.1:5672-127.0.0.1:39918 user=admin@QPID exName=qmf.default.topic qName=qmfc-v2-ui-d64.3775.1 key=agent.ind.event.# args={}

root@d64:~# qpid-tool     admin/1qazs@localhost:5672  
Management Tool for QPID
qpid: list
Summary of Objects by Type:
    Package                 Class         Active  Deleted
    =======================================================
    org.apache.qpid.broker  binding       12      12
    org.apache.qpid.broker  broker        1       0
    org.apache.qpid.broker  system        1       0
    org.apache.qpid.broker  subscription  5       5
    org.apache.qpid.broker  connection    1       1
    org.apache.qpid.broker  session       1       1
    org.apache.qpid.broker  queue         5       5
    org.apache.qpid.broker  exchange      8       0
    org.apache.qpid.broker  vhost         1       0

root@d64:~# qpid-stat -c admin/1qazs@localhost:5672
Connections
  client-addr                     cproc      cpid  auth        connected  idle  msgIn  msgOut
  =============================================================================================
  127.0.0.1:5672-127.0.0.1:39928  qpid-stat  3969  admin@QPID  2s         0s     251    320




qpid modules location, problems with loading Qpid plugins/extensions:
/usr/lib64/qpid/daemon/


lsof -n -p $(pgrep qpidd)


 # openssl pkcs12 -export -out  os-mysql1.local.p12 -inkey os-mysql1.local.key -in os-mysql1.local.crt 
 11 # permissions to read cert file 
 12 # [root@os-mysql1 ha_qpid]# certutil -L -d /etc/pki/qpidd/
 13 # certutil: function failed: SEC_ERROR_LEGACY_DATABASE: The certificate/key database is in an old, unsupported format.
 14 # sudo -u qpidd /usr/sbin/qpidd --config /etc/qpidd.conf
 15 #
 16 #   $ certutil -L -d non-existent
 17 #       certutil: function failed: SEC_ERROR_LEGACY_DATABASE: The certificate/key database is in an old, unsupported format.

 Jun 22 11:10:43 qpid1 qpidd[739]: 2014-06-22 11:10:43 error Failed to initialise SSL plugin: Failed: NSS error [-8015] (qpid/sys/ssl/util.cpp:103)