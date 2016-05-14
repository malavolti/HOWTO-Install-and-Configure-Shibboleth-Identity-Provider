# HOWTO Install and Configure a Shibboleth IdP v3.2.1 on Ubuntu Linux LTS 16.04 with Apache2 + Tomcat8

## Table of Contents

1. [Requirements Hardware](#requirements-hardware)
2. [Requirements Software](#requirements-software)
3. [Other Requirements](#other-requirements)
4. [Installation Instructions](#installation-instructions)
  1. [Install software requirements](#install-software-requirements)
  2. [Configure the environment](#configure-the-environment)
5. [Configuration Instructions](#configuration-instructions)
  1. [Configure Apache Tomcat 8](#configure-apache-tomcat-8)
  2. [Configure Shibboleth Identity Provider v3.2.1 to release the persistent-id (Stored Mode)](#configure-shibboleth-identity-provider-v321-to-release-the-persistent-id-stored-mode)
  3. [Configure Attribute Filter for Research and Scholarship Entity Category](#configure-attribute-filter-for-research-and-scholarship-entity-category)


## Requirements Hardware

 * CPU: 2 Core
 * RAM: 4 GB
 * HDD: 20 GB

## Requirements Software

 * ca-certificates
 * ntp
 * default-jdk 
 * tomcat8
 * apache2

## Other Requirements

 * Place the HTTPS Server Certificate and the HTTTPS Server Key inside ```/tmp``` directory

## Installation Instructions

### Install software requirements

0. Become ROOT of the machine: 
  * ```sudo su -```

1. Install GIT and retrieve this repository: 
  * ```apt-get install git```
  * ```git clone https://github.com/malavolti/HOWTO-Install-and-Configure-Shibboleth-Identity-Provider.git /usr/local/src/HOWTO-Shib-IdP```

2. Install the packages required: 
  * ```apt-get install vim default-jdk ca-certificates openssl tomcat8 apache2 ntp```

### Configure the environment

0. Modify your ```/etc/hosts```:
  * ```vim /etc/hosts```
  
    ```bash
    127.0.1.1 idp.example.org idp
    ```

1. Be sure that your firewall doesn't block the traffic on port **443** (or you can't access to your IdP)

2. Define the global costants ```JAVA_HOME``` and ```IDP_SRC``` inside ```/etc/environment```:
  * ```vim /etc/environment```
  
    ```bash
    JAVA_HOME=/usr/lib/jvm/java-8-openjdk-amd64/jre
    IDP_SRC=/usr/local/src/shibboleth-identity-provider-3.2.1
    ```

  * ```source /etc/environment```
  * ```export JAVA_HOME=/usr/lib/jvm/java-8-openjdk-amd64/jre```
  * ```export IDP_SRC=/usr/local/src/shibboleth-identity-provider-3.2.1```
  
3. Move the Certificate and the Key file for HTTPS server from ```/tmp/``` to ```/root/certificates```:
  * ```mkdir /root/certificates```
  * ```mv /tmp/idp-cert-server.crt /root/certificates```
  * ```mv /tmp/idp-key-server.key /root/certificates```
  * ```chmod 400 /root/certificates/idp-key-server.key```
  * ```chmod 644 /root/certificates/idp-cert-server.crt```
  
  (OPTIONAL) Create a Certificate and a Key self-signed if you don't have the official ones provided by DigiCert:
  * ```openssl req -x509 -newkey rsa:4096 -keyout /root/certificates/idpkey-server.key -out /root/certificates/idp-cert-server.crt -nodes -days 3650```

4. Configure **/etc/default/tomcat8**:
  * ```update-alternatives --config java``` (copy the path without /bin/java)
  * ```update-alternatives --config javac```
  * ```vim /etc/default/tomcat8```
  
    ```bash
    JAVA_HOME=/usr/lib/jvm/java-8-openjdk-amd64/jre
    ...
    JAVA_OPTS="-Djava.awt.headless=true -XX:+DisableExplicitGC -XX:+UseParallelOldGC -Xms256m -Xmx2g -Djava.security.egd=file:/dev/./urandom"
    ```

### Install Shibboleth Identity Provider 3.2.1

0. Become ROOT of the machine: 
  * ```sudo su -```

1. Download the Shibboleth IdP 3.2.1:
  * ```cd /usr/local/src``
  * ```wget https://shibboleth.net/downloads/identityprovider/latest/shibboleth-identity-provider-3.2.1.tar.gz```
  * ```tar -xzvf shibboleth-identity-provider-3.2.1.tar.gz```
  * ```cd shibboleth-identity-provider-3.2.1```

2. Run the installer ```install.sh```:
  * ```./bin/install.sh```
  
  ```bash
  **root@idp:/usr/local/src/shibboleth-identity-provider-3.2.1#** ./bin/install.sh
  Source (Distribution) Directory: [/usr/local/src/shibboleth-identity-provider-3.2.1]
  Installation Directory: [/opt/shibboleth-idp]
  Hostname: [localhost.localdomain]
  idp.example.it
  SAML EntityID: [https://idp.example.it/idp/shibboleth]
  Attribute Scope: [localdomain]
  example.it
  Backchannel PKCS12 Password: ###PASSWORD-FOR-BACKCHANNEL###
  Re-enter password: ###PASSWORD-FOR-BACKCHANNEL###
  Cookie Encryption Key Password: ###PASSWORD-FOR-COOKIE-ENCRYPTION###
  Re-enter password: ###PASSWORD-FOR-COOKIE-ENCRYPTION###
  ```
  
  From this point the variable **idp.home** refers to the directory: ```/opt/shibboleth-idp```

3. Import the libraries JST to visualize the IdP ```status``` page:
  * ```cd /opt/shibboleth-idp/edit-webapp/WEB-INF/lib```
  * ```wget https://build.shibboleth.net/nexus/service/local/repositories/thirdparty/content/javax/servlet/jstl/1.2/jstl-1.2.jar```
  * ```cd /opt/shibboleth-idp/bin ; ./build.sh -Didp.target.dir=/opt/shibboleth-idp```

4. Change the rights to enable tomcat8 user to access on the following directories:
  * ```chown -R tomcat8/opt/shibboleth-idp/logs/```
  * ```chown -R tomcat8 /opt/shibboleth-idp/metadata/```
  * ```chown -R tomcat8 /opt/shibboleth-idp/credentials/```
  * ```chown -R tomcat8 /opt/shibboleth-idp/conf/```

## Configuration Instructions

### Configure SSL on Apache2 (Tomcat8 front-end)

1. Modify the file ```/etc/apache2/sites-available/default-ssl.conf``` as follows:

```apache
<VirtualHost _default_:443>
 ServerName idp.example.garr.it:443
 ServerAdmin admin@example.garr.it
 DocumentRoot /var/www/html
 ...
 SSLEngine On
 SSLProtocol all -SSLv2 -SSLv3 -TLSv1
 SSLCipherSuite "kEDH+AESGCM:ECDHE-RSA-CHACHA20-POLY1305:ECDHE-RSA-AES256-SHA384:ECDHE-RSA-AES256-SHA256:ECDHE-RSA-AES256-SHA:ECDHE-RSA-AES256-GCMSHA384:ECDHE-RSA-AES256-GCM-SHA256:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-ECDSAAES256-SHA384:ECDHE-ECDSA-AES256-SHA256:ECDHE-ECDSA-AES256-SHA:ECDHE-ECDSAAES256-GCM-SHA384:ECDHE-ECDSA-AES256-GCM-SHA256:AES256-GCM-SHA384:!3DES:!DES:!DHE-RSA-AES128-GCM-SHA256:!DHE-RSA-AES256-SHA:!EDE3:!EDH-DSS-CBC-SHA:!EDH-DSSDES-CBC3-SHA:!EDH-RSA-DES-CBC-SHA:!EDH-RSA-DES-CBC3-SHA:!EXP-EDH-DSS-DES-CBCSHA:!EXP-EDH-RSA-DES-CBC-SHA:!EXPORT:!MD5:!PSK:!RC4-SHA:!aNULL:!eNULL"
 SSLHonorCipherOrder on
 # Disable SSL Compression
 SSLCompression Off
 # Enable HTTP Strict Transport Security with a 2 year duration
 Header always set Strict-Transport-Security "max-age=63072000;includeSubDomains"
 ...
 SSLCertificateFile /root/certificates/idp-cert-server.pem
 SSLCertificateKeyFile /root/certificates/idp-key-server.pem
 ...
</VirtualHost>
```

2. Enable **SSL** and "headers" Apache2 modules:
* ```a2enmod ssl headers```
* ```a2ensite default-ssl.conf```
* ```a2dissite 000-default.conf```
* ```service apache2 restart```

3. Configure Apache2 to open port 80 only for localhost:
* ```vim /etc/apache2/ports.conf```

  ```apache
  # If you just change the port or add more ports here, you will likely also
  # have to change the VirtualHost statement in
  # /etc/apache2/sites-enabled/000-default.conf
  
  Listen 127.0.0.1:80

  <IfModule ssl_module>
    Listen 443
  </IfModule>
  <IfModule mod_gnutls.c>
    Listen 443
  </IfModule>
  ```
  
4. Verify the strength of your IdP's machine on:
  * [**https://www.ssllabs.com/ssltest/analyze.html**](https://www.ssllabs.com/ssltest/analyze.html)

### Configure Tomcat 8

0. Become ROOT of the machine: 
  * ```sudo su -```

1. Change "server.xml":
  * ```vim /etc/tomcat8/server.xml```
  
    Comment out the Connector 8080 (HTTP):
    
    ```tomcat
    <!-- A "Connector" represents an endpoint by which requests are received
         and responses are returned. Documentation at :
         Java HTTP Connector: /docs/config/http.html (blocking & non-blocking)
         Java AJP  Connector: /docs/config/ajp.html
         APR (HTTP/AJP) Connector: /docs/apr.html
         Define a non-SSL/TLS HTTP/1.1 Connector on port 8080
    -->
    <!--
    <Connector port="8080" protocol="HTTP/1.1"
               connectionTimeout="20000"
               URIEncoding="UTF-8"
               redirectPort="8443" />
    -->
    ```
    
    Enable the Connector 8009 (AJP):
    
    ```tomcat
    <!-- Define an AJP 1.3 Connector on port 8009 -->
    <Connector port="8009" protocol="AJP/1.3" redirectPort="443" address="127.0.0.1" enableLookups="false" tomcatAuthentication="false"/>
    ```
    
2. Create and change the file ```idp.xml```:
  * ```sudo vim /etc/tomcat8/Catalina/localhost/idp.xml```
    ```tomcat
    <Context docBase="/opt/shibboleth-idp/war/idp.war"
             privileged="true"
             antiResourceLocking="false"
             swallowOutput="true"/>
    ```

3. Create the Apache2 configuration file for IdP:

  * ```vim /etc/apache2/sites-available/idp.conf```
  
  ```apache
  <Proxy ajp://localhost:8009>
    Require all granted
  </Proxy>
  
  ProxyPass /idp ajp://localhost:8009/idp retry=5
  ProxyPassReverse /idp ajp://localhost:8009/idp retry=5
  ```

4. Enable proxy_ajp apache2 module and the new IdP site:
  * ```a2enmod proxy_ajp ; a2ensite idp.conf ; service apache2 restart```
  
5. Modify context.xml to prevent error of **lack of persistence of the session objects* created by the IdP :
  * ```vim /etc/tomcat8/context.xml```

    and remove the comment from:

    ```<Manager pathname="" />```
    
6. Verify if the IdP works by opening this page on your browser:
  * ```https://idp.example.it/idp/shibboleth``` (you should see the IdP metadata)
  
### Speed up Tomcat 8 startup

0. Become ROOT of the machine: 
  * ```sudo su -```
  
1. Find out the JARs that can be skipped from the scanning:
  * ```cd /opt/shibboleth-idp/```
  * ```ls webapp/WEB-INF/lib | awk '{print $1",\\"}'```
  
2. Insert the output list into ```/opt/tomcat/conf/catalina.properties``` at the tail of ```tomcat.util.scan.StandardJarScanFilter.jarsToSkip```

3. Restart Tomcat 8:
  * ```service tomcat8 restart```
  
### Configure Shibboleth IdP v3.2.1

0. Become ROOT of the machine: 
  * ```sudo su -```
  
1. Test IdP by opening a terminal and running these commands:
  * ```cd /opt/shibboleth-idp/bin```
  * ```./status.sh``` (You shuold see some informations about the IdP installed)

2. Install a MySQL database and import the libraries used by Tomcat and Shibboleth:
  * ```apt-get istall mysql-server libmysql-java```
  * ```cp /usr/share/java/mysql-connector-java.jar /opt/shibboleth-idp/editwebapp/WEB-INF/lib/```
  * ```cp /usr/share/java/mysql-connector-java.jar /usr/share/tomcat8/lib/```
  * ```cp /usr/share/tomcat8/lib/tomcat-jdbc.jar /opt/shibboleth-idp/editwebapp/WEB-INF/lib/```

3. Install the libraries **Common DBCP2** used for generation of saml-id:
  * ```cd /usr/local/src/```
  * ```wget http://mirrors.muzzy.it/apache//commons/dbcp/binaries/commonsdbcp2-2.1.1-bin.tar.gz```
  * ```tar xzvf commons-dbcp2-2.1.1-bin.tar.gz ; cd commons-dbcp2-2.1.1/```
  * ```cp commons-dbcp2-2.1.1.jar /opt/shibboleth-idp/edit-webapp/WEBINF/lib/```
  
4. Install the libraries Tomcat Common Pool used for the generation of saml-id:
  * ```cd /usr/local/src/```
  * ``` ```
  * ``` ```
  * ``` ```
  * ``` ```
  
    
