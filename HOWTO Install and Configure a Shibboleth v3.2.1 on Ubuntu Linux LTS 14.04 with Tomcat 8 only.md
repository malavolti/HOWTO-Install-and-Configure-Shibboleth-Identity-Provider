# HOWTO Install and Configure a Shibboleth IdP v3.2.1 on Ubuntu Linux LTS 14.04 with Tomcat 8 only

## Table of Contents

* [Requiremets Hardware](#requirements-hardware)
* [Requiremets Software](#requirements-software)
* [Other Requiremets](#other-requirements)
* [Installation Instruction](#installation-instructions)
  * [Install software requirements](#install-software-requirements)

## Requirements Hardware

 * CPU: 2 Core
 * RAM: 4 GB
 * HDD: 20 GB

## Requirements Software

 * ca-certificates
 * ntp
 * openSSL >= 1.0.2
 * oracle-java8-installer 
 * oracle-java8-unlimited-jce-policy
 * Tomcat 8
 * Shibboleth Identity Provider

## Other Requirements

 * Place the HTTPS Server Certificate and the HTTTPS Server Key into the ```/tmp``` directory

## Installation Instructions

### Install software requirements

1. Become ROOT of the machine: 
   * ```sudo su -```

2. Install the packages required: 
   * ```apt-get install vim ntp```

3. Install Sun Java 8 Oracle:
   * ```apt-get install python-software-properties```
   * ```add-apt-repository ppa:webupd8team/java```
   * ```apt-get update```
   * ```apt-get install oracle-java8-installer oracle-java8-unlimited-jce-policy```

4. Configure the Sun Java Oracle:
   * ```update-alternatives --config java``` (copy the path without the ```/bin/java``` part)
   * ```update-alternatives --config javac```
   * ```vim /etc/environment```

5. Modify the machine environment: ``` vim /etc/environment ```
   ```bash
   JAVA_HOME=##PATH_COPIED##
   CATALINA_HOME=/opt/tomcat
   ```

6. Applied changes:
   * ```source /etc/environment```

7. Verify the changes applied:
   ```bash
   echo "JAVA_HOME="$JAVA_HOME ; echo "CATALINA_HOME="$CATALINA_HOME
   ```

8. Install openSSL >= 1.0.2:
   * ```apt-get install make ant gcc libapr1-dev libssl-dev```
   * ```cd /usr/local/src```
   * ```wget https://www.openssl.org/source/openssl-1.0.2g.tar.gz```
   * ```tar -xvzf openssl-1.0.2g.tar.gz```
   * ```cd openssl-1.0.2g/```
   * ```./config --prefix=/usr/```
   * ```make depend```
   * ```make && make install```
   * ```openssl version``` (useful to verify the installation)

9. Install Apache APR (Apache Portable Runtime):
   * ```cd /usr/local/src```
   * ```wget http://mirror.nohup.it/apache//apr/apr-1.5.2.tar.gz```
   * ```tar -xvzf apr-1.5.2.tar.gz```
   * ```cd apr-1.5.2 ; vim configure```
   * change the line **30206** by replace the value of ```$RM "$cfgfile"``` with ```"$RM" -f "$cfgfile"```
   * ```./configure```
   * ```make```
   * ```make test```
   * ```make install```

   The library APR will be installed into ```/usr/local/apr/lib``` directory
   
10. Install Apache Tomcat 8:
    * ```cd /usr/local/src/```
    * ```wget http://apache.panu.it/tomcat/tomcat-8/v8.0.33/bin/apache-tomcat-8.0.33.tar.gz```
    * ```mkdir /opt/tomcat```
    * ```tar xvf apache-tomcat-8*tar.gz -C /opt/tomcat --strip-components=1```
    * ```groupadd tomcat```
    * ```useradd -s /bin/false -g tomcat -d /opt/tomcat tomcat```
    * ```cd /opt/tomcat```
    * ```chgrp -R tomcat conf```
    * ```chmod g+rwx conf```
    * ```chmod g+r conf/*```

11. Create the init.d service for restarting Tomcat 8:
   * ```vim /etc/init.d/tomcat``` (and paste the content of the [daemon-tomcat](../blob/master/daemon-tomcat) file)
   * ```cd /usr/local/src```
   * ```chmod 755 /etc/init.d/tomcat```
   * ```update-rc.d tomcat defaults```
   * ```service tomcat start```

12. Ensure that your firewall/firewalls are not blocking the traffic on the port **443** and **8080**:

13. Open the Tomcat 8 Home Page (fix the URL before open on your browser):
   * ```http://idp.example.garr.it:8080``` and verify to see the Tomcat 8 Home Page

14. Move the HTTPS Certificate and Key to a new directory ```/opt/tomcat/ssl```:
   * ```mkdir /opt/tomcat/ssl```
   * ```mv /tmp/cert-server.pem /opt/tomcat/ssl/```
   * ```mv /tmp/key-server.pem /opt/tomcat/ssl/```
   * ```chmod 400 /opt/tomcat/ssl/key-server.pem```

15. Install the Tomcat Native:
   * ```cd /usr/local/src```
   * ```wget http://apache.panu.it/tomcat/tomcatconnectors/native/1.2.5/source/tomcat-native-1.2.5-src.tar.gz```
   * ```tar -xvzf tomcat-native-*-src.tar.gz```
   * ```cd /usr/local/src/tomcat-native-*-src/```
   * ```ant```
   * ```cd /usr/local/src/tomcat-native-*-src/native```
   * ```./configure --with-apr=/usr/local/apr --with-javahome=/usr/lib/jvm/java-8-oracle --with-ssl=yes --prefix=/opt/tomcat```
   * ```make && make install```

16. Create Tomcat 8 environment: 
   * ```touch /opt/tomcat/bin/setenv.sh ; vim touch /opt/tomcat/bin/setenv.sh```
   ```bash
   export LD_LIBRARY_PATH="$LD_LIBRARY_PATH:/usr/local/apr/lib"
   export LD_LIBRARY_PATH="$LD_LIBRARY_PATH:/opt/tomcat/lib"
   ```
   * ```service tomcat restart```

17. Enable the port **443** for the HTTPS by changing the **8443** connector into this one:
  ```xml
  ...
  <!-- Define a SSL/TLS HTTP/1.1 Connector on port 443
  This connector uses the NIO implementation that requires the JSSE
  style configuration. When using the APR/native implementation, the
  OpenSSL style configuration is required as described in the APR/native
  documentation -->
  <Connector
      protocol="org.apache.coyote.http11.Http11AprProtocol"
      port="443" maxThreads="200" maxPostSize="100000"
      scheme="https" secure="true" SSLEnabled="true"
      SSLCertificateFile="/opt/tomcat/ssl/cert-server.pem"
      SSLCertificateKeyFile="/opt/tomcat/ssl/key-server.pem"
      SSLCertificateChainFile="/opt/tomcat/ssl/Terena-chain.pem"
      SSLVerifyClient="optional" SSLProtocol="TLSv1+TLSv1.1+TLSv1.2"/>
  ...
  ```
   * ```service tomcat restart```

18. Manage Tomcat with its GUI:
   * ```sudo su -```
   * ```vim $CATALINA_HOME/conf/tomcat-users.xml```
   ```xml
   <tomcat-users>
      ...
      <role rolename="manager-gui"/>
      <role rolename="admin-gui"/>
      <user username="admin" password="**password_administrator**" roles="admin-gui,manager-gui"/>
      <user username="manager" password="**password_manager**" roles="manager-gui"/>
    </tomcat-users>
    ```
  * Try to login on: https://idp.example.garr.it/manager/html with the user "**manager**" and remove all application deployed not directly involved with the IdP to improve the speed of Tomcat loading.
