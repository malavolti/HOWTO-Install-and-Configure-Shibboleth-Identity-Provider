# HOWTO Install and Configure a Shibboleth IdP v3.3.2 on Ubuntu Linux LTS 16.04 with Apache2 + Tomcat8

## Table of Contents

1. [Requirements Hardware](#requirements-hardware)
2. [Requirements Software](#requirements-software)
3. [Other Requirements](#other-requirements)
4. [Installation Instructions](#installation-instructions)
  1. [Install software requirements](#install-software-requirements)
  2. [Configure the environment](#configure-the-environment)
  3. [Install Shibboleth Identity Provider 3.3.2](#install-shibboleth-identity-provider-v332)
5. [Configuration Instructions](#configuration-instructions)
  1. [Configure SSL on Apache2 (Tomcat8 front-end)](#configure-ssl-on-apache2-tomcat8-front-end)
  2. [Configure Apache Tomcat 8](#configure-apache-tomcat-8)
  3. [Speed up Tomcat 8 startup](#speed-up-tomcat-8-startup)
  4. [Configure Shibboleth Identity Provider v3.3.2 to release the persistent-id (Stored Mode)](#configure-shibboleth-identity-provider-v332-to-release-the-persistent-id-stored-mode)
  5. [Configure Attribute Filter for Research and Scholarship Entity Category](#configure-attribute-filter-for-research-and-scholarship-entity-category)


## Requirements Hardware

 * CPU: 2 Core
 * RAM: 4 GB
 * HDD: 20 GB

## Requirements Software

 * ca-certificates
 * ntp
 * vim
 * default-jdk 
 * tomcat8
 * apache2

## Other Requirements

 * Place your TLS certificates inside ```/tmp``` directory

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

1. Be sure that your firewall **doesn't block** the traffic on port **443** (or you can't access to your IdP)

2. Define the costants ```JAVA_HOME``` and ```IDP_SRC``` inside ```/etc/environment```:
  * ```vim /etc/environment```

    ```bash
    JAVA_HOME=/usr/lib/jvm/java-8-openjdk-amd64/jre
    IDP_SRC=/usr/local/src/shibboleth-identity-provider-3.3.2
    ```
  * ```source /etc/environment```
  * ```export JAVA_HOME=/usr/lib/jvm/java-8-openjdk-amd64/jre```
  * ```export IDP_SRC=/usr/local/src/shibboleth-identity-provider-3.3.2```
  
3. Move the Certificate and the Key file for HTTPS server from ```/tmp/``` to ```/root/certificates```:
  * ```mkdir /root/certificates```
  * ```mv /tmp/idp-cert-server.pem /root/certificates```
  * ```mv /tmp/idp-key-server.pem /root/certificates```
  * ```chmod 400 /root/certificates/idp-key-server.pem```
  * ```chmod 644 /root/certificates/idp-cert-server.pem```
  
  (OPTIONAL) Create a Certificate and a Key self-signed if you don't have the official ones provided by DigiCert:
  * ```openssl req -x509 -newkey rsa:4096 -keyout /root/certificates/idpkey-server.pem -out /root/certificates/idp-cert-server.pem -nodes -days 3650```

4. Configure **/etc/default/tomcat8**:
  * ```update-alternatives --config java``` (copy the path without /bin/java)
  * ```update-alternatives --config javac```
  * ```vim /etc/default/tomcat8```
  
    ```bash
    JAVA_HOME=/usr/lib/jvm/java-8-openjdk-amd64/jre
    ...
    JAVA_OPTS="-Djava.awt.headless=true -XX:+DisableExplicitGC -XX:+UseParallelOldGC -Xms256m -Xmx2g -Djava.security.egd=file:/dev/./urandom"
    ```

    (This settings configure the memory of the JVM that will host the IdP Web Application. 
    The Memory value depends on the phisical memory installed on the machine. 
    Set the "**Xmx**" (max heap space available to the JVM) at least to **2GB**)

### Install Shibboleth Identity Provider v3.3.2

0. Become ROOT of the machine: 
  * ```sudo su -```

1. Download the Shibboleth IdP 3.3.2:
  * ```cd /usr/local/src```
  * ```wget http://shibboleth.net/downloads/identity-provider/latest/shibboleth-identity-provider-3.3.2.tar.gz```
  * ```tar -xzvf shibboleth-identity-provider-3.3.2.tar.gz```
  * ```cd shibboleth-identity-provider-3.3.2```

2. Run the installer ```install.sh```:
  * ```./bin/install.sh```
  
  ```bash
  root@idp:/usr/local/src/shibboleth-identity-provider-3.3.2# ./bin/install.sh
  Source (Distribution) Directory: [/usr/local/src/shibboleth-identity-provider-3.3.2]
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

3. Import the JST libraries to visualize the IdP ```status``` page:
  * ```cd /opt/shibboleth-idp/edit-webapp/WEB-INF/lib```
  * ```wget https://build.shibboleth.net/nexus/service/local/repositories/thirdparty/content/javax/servlet/jstl/1.2/jstl-1.2.jar```
  * ```cd /opt/shibboleth-idp/bin ; ./build.sh -Didp.target.dir=/opt/shibboleth-idp```

4. Change the rights to enable **tomcat8** user to access on the following directories:
  * ```chown -R tomcat8 /opt/shibboleth-idp/logs/```
  * ```chown -R tomcat8 /opt/shibboleth-idp/metadata/```
  * ```chown -R tomcat8 /opt/shibboleth-idp/credentials/```
  * ```chown -R tomcat8 /opt/shibboleth-idp/conf/```

## Configuration Instructions

### Configure SSL on Apache2 (Tomcat8 front-end)

1. Modify the file ```/etc/apache2/sites-available/default-ssl.conf``` as follows:

  ```apache
  <VirtualHost _default_:443>
    ServerName idp.example.it:443
    ServerAdmin admin@example.it
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

2. Enable **SSL** and **headers** Apache2 modules:
  * ```a2enmod ssl headers```
  * ```a2ensite default-ssl.conf```
  * ```a2dissite 000-default.conf```
  * ```systemctl restart apache2```

3. Configure Apache2 to open port **80** only for localhost:
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

### Configure Apache Tomcat 8

0. Become ROOT of the machine: 
  * ```sudo su -```

1. Change ```server.xml```:
  * ```vim /etc/tomcat8/server.xml```
  
    Comment out the Connector 8080 (HTTP):
    
    ```apache
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

    ```apache
    <!-- Define an AJP 1.3 Connector on port 8009 -->
    <Connector port="8009" protocol="AJP/1.3" redirectPort="443" address="127.0.0.1" enableLookups="false" tomcatAuthentication="false"/>
    ```

2. Create and change the file ```idp.xml```:
  * ```sudo vim /etc/tomcat8/Catalina/localhost/idp.xml```

    ```apache
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

4. Enable **proxy_ajp** apache2 module and the new IdP site:
  * ```a2enmod proxy_ajp ; a2ensite idp.conf ; systemctl restart apache2```
  
5. Modify **context.xml** to prevent error of *lack of persistence of the session objects* created by the IdP :
  * ```vim /etc/tomcat8/context.xml```

    and remove the comment from:

    ```<Manager pathname="" />```
6. Restart **tomcat8** to apply config changes:
  * ```systemctl restart tomcat8```

7. Verify if the IdP works by opening this page on your browser:
  * ```https://idp.example.it/idp/shibboleth``` (you should see the IdP metadata)
  
### Speed up Tomcat 8 startup

0. Become ROOT of the machine: 
  * ```sudo su -```
  
1. Find out the JARs that can be skipped from the scanning:
  * ```cd /opt/shibboleth-idp/```
  * ```ls webapp/WEB-INF/lib | awk '{print $1",\\"}'```
  
2. Insert the output list into ```/opt/tomcat/conf/catalina.properties``` at the tail of ```tomcat.util.scan.StandardJarScanFilter.jarsToSkip```

3. Restart Tomcat 8:
  * ```systemctl restart tomcat8```
  
### Configure Shibboleth Identity Provider v3.3.2 to release the persistent-id (Stored mode)

0. Become ROOT of the machine: 
  * ```sudo su -```
  
1. Test IdP by opening a terminal and running these commands:
  * ```cd /opt/shibboleth-idp/bin```
  * ```./status.sh``` (You should see some informations about the IdP installed)

2. Install **MySQL Connector Java** and **Tomcat JDBC** libraries used by Tomcat and Shibboleth for MySQL DB:
  * ```apt-get install mysql-server libmysql-java```
  * ```cp /usr/share/java/mysql-connector-java.jar /opt/shibboleth-idp/edit-webapp/WEB-INF/lib/```
  * ```cp /usr/share/java/mysql-connector-java.jar /usr/share/tomcat8/lib/```
  * ```cp /usr/share/tomcat8/lib/tomcat-jdbc.jar /opt/shibboleth-idp/edit-webapp/WEB-INF/lib/```

3. Install the libraries **Common DBCP2**[[2]](http://commons.apache.org/proper/commons-dbcp/) used for generation of saml-id:
  * ```cd /usr/local/src/```
  * ```wget http://www-us.apache.org/dist//commons/dbcp/binaries/commons-dbcp2-2.2.0-bin.tar.gz```
  * ```tar xzvf commons-dbcp2-2.2.0-bin.tar.gz ; cd commons-dbcp2-2.2.0/```
  * ```cp commons-dbcp2-2.2.0.jar /opt/shibboleth-idp/edit-webapp/WEB-INF/lib/```
  
4. Install the libraries **Tomcat Common Pool**[[3]](http://commons.apache.org/proper/commons-pool/download_pool.cgi) used for the generation of saml-id:
  * ```cd /usr/local/src/```
  * ```wget http://mirror.nohup.it/apache//commons/pool/binaries/commons-pool2-2.5.0-bin.tar.gz```
  * ```tar xzvf commons-pool2-2.5.0-bin.tar.gz ; cd commons-pool2-2.5.0/```
  * ```cp commons-pool2-2.5.0.jar /opt/shibboleth-idp/edit-webapp/WEB-INF/lib/```

5. Rebuild the **idp.war** of Shibboleth with the new libraries:
  * ```cd /opt/shibboleth-idp/ ; ./bin/build.sh```

6. Create and prepare the "**shibboleth**" MySQL DB to host the values of the several **persistent-id** and **StorageRecords** to host other useful information about user consent:
  *  ```cd /usr/local/src/HOWTO-Shib-IdP```
  *  Modify the [shibboleth-db.sql](../master/shibboleth-db.sql) by changing the *username* and *password* of the user that has access to the "**shibboleth**" DB.
  *  ```mysql -u root -p < ./shibboleth-db.sql```
  *  ```systemctl restart mysql```

7. Enable the generation of the ```persistent-id``` (this replace the deprecated attribute *eduPersonTargetedID*)
  * ```vim /opt/shibboleth-idp/conf/saml-nameid.properties```
    (the *sourceAttribute* MUST BE an attribute, or a list of comma-separated attributes, that uniquely identify the subject of the generated ```persistent-id```. It MUST BE: **Stable**, **Permanent** and **Not-reassignable**)

    ```xml
    idp.persistentId.sourceAttribute = uid
    ...
    idp.persistentId.salt = ### result of 'openssl rand -base64 36'###
    ...
    idp.persistentId.generator = shibboleth.StoredPersistentIdGenerator
    ...
    idp.persistentId.dataSource = MyDataSource
    ...
    idp.persistentId.computed = shibboleth.ComputedPersistentIdGenerator
    ```

  * Enable the **SAML2PersistentGenerator**:
    * ```vim /opt/shibboleth-idp/conf/saml-nameid.xml```
      * Remove the comment from the line containing:

        ```xml
        <ref bean="shibboleth.SAML2PersistentGenerator" />
        ```

      * ```vim /opt/shibboleth-idp/conf/c14n/subject-c14n.xml```
        * Remove the comment to the bean called "**c14n/SAML2Persistent**".

8. Enable **JPAStorageService** for the **StorageService** of the user consent:
  * ```vim /opt/shibboleth-idp/conf/global.xml``` and add to the tail of the file this piece code:

    ```xml
    <!-- A DataSource bean suitable for use in the idp.persistentId.dataSource property. -->
    <bean id="MyDataSource" class="org.apache.commons.dbcp.BasicDataSource"
          p:driverClassName="com.mysql.jdbc.Driver"
          p:url="jdbc:mysql://localhost:3306/shibboleth?autoReconnect=true"
          p:username="##USER_DB##"
          p:password="##PASSWORD##"
          p:maxActive="10"
          p:maxIdle="5"
          p:maxWait="15000"
          p:testOnBorrow="true"
          p:validationQuery="select 1"
          p:validationQueryTimeout="5" />

    <bean id="shibboleth.JPAStorageService" class="org.opensaml.storage.impl.JPAStorageService"
          p:cleanupInterval="%{idp.storage.cleanupInterval:PT10M}"
          c:factory-ref="shibboleth.JPAStorageService.entityManagerFactory"/>

    <bean id="shibboleth.JPAStorageService.entityManagerFactory"
          class="org.springframework.orm.jpa.LocalContainerEntityManagerFactoryBean">
          <property name="packagesToScan" value="org.opensaml.storage.impl"/>
          <property name="dataSource" ref="MyDataSource"/>
          <property name="jpaVendorAdapter" ref="shibboleth.JPAStorageService.JPAVendorAdapter"/>
          <property name="jpaDialect">
            <bean class="org.springframework.orm.jpa.vendor.HibernateJpaDialect" />
          </property>
    </bean>

    <bean id="shibboleth.JPAStorageService.JPAVendorAdapter" class="org.springframework.orm.jpa.vendor.HibernateJpaVendorAdapter">
            <property name="database" value="MYSQL" />
    </bean>
    ```
    (and modify the "**USER_DB_NAME**" and "**PASSWORD**" of the "**shibboleth**" DB)

  * Modify the IdP configuration file:
    * ```vim /opt/shibboleth-idp/conf/idp.properties```

      ```xml
      idp.session.StorageService = shibboleth.JPAStorageService
      idp.consent.StorageService = shibboleth.JPAStorageService
      idp.replayCache.StorageService = shibboleth.JPAStorageService
      idp.artifact.StorageService = shibboleth.JPAStorageService
      ```
  
      (This will say to IdP to store the data collected by User Consent into the "**StorageRecords**" table)

9. Connect the openLDAP to the IdP to allow the authentication of the users:
  * ```vim /opt/shibboleth-idp/conf/ldap.properties```

    (with ***TLS** solutions we consider to have the LDAP certificate into ```/opt/shibboleth-idp/credentials```).

    *  Solution 1: LDAP + STARTTLS:

      ```xml
      idp.authn.LDAP.authenticator = bindSearchAuthenticator
      idp.authn.LDAP.ldapURL = ldap://ldap.example.it:389
      idp.authn.LDAP.useStartTLS = true
      idp.authn.LDAP.useSSL = false
      idp.authn.LDAP.sslConfig = certificateTrust
      idp.authn.LDAP.trustCertificates = %{idp.home}/credentials/ldap-server.crt
      idp.authn.LDAP.baseDN = ou=people,dc=example,dc=it
      idp.authn.LDAP.userFilter = (uid={user})
      idp.authn.LDAP.bindDN = cn=admin,dc=example,dc=it
      idp.authn.LDAP.bindDNCredential = ###LDAP ADMIN PASSWORD###
      ```

    * Solution 2: LDAP + TLS:

      ```xml
      idp.authn.LDAP.authenticator = bindSearchAuthenticator
      idp.authn.LDAP.ldapURL = ldaps://ldap.example.it:636
      idp.authn.LDAP.useStartTLS = false
      idp.authn.LDAP.useSSL = true
      idp.authn.LDAP.sslConfig = certificateTrust
      idp.authn.LDAP.trustCertificates = %{idp.home}/credentials/ldap-server.crt
      idp.authn.LDAP.baseDN = ou=people,dc=example,dc=it
      idp.authn.LDAP.userFilter = (uid={user})
      idp.authn.LDAP.bindDN = cn=admin,dc=example,dc=it
      idp.authn.LDAP.bindDNCredential = ###LDAP ADMIN PASSWORD###
      ```

    * Solution 3: plain LDAP
  
      ```xml
      idp.authn.LDAP.authenticator = bindSearchAuthenticator
      idp.authn.LDAP.ldapURL = ldap://ldap.example.it:389
      idp.authn.LDAP.useStartTLS = false
      idp.authn.LDAP.useSSL = false
      idp.authn.LDAP.baseDN = ou=people,dc=example,dc=it
      idp.authn.LDAP.userFilter = (uid={user})
      idp.authn.LDAP.bindDN = cn=admin,dc=example,dc=it
      idp.authn.LDAP.bindDNCredential = ###LDAP ADMIN PASSWORD###
      ```
      (If you decide to use the Solution 3, you have to remove the following code from your ```attribute-resolver-full.xml```:
      
      ```xml
      </dc:FilterTemplate>
      <!--
      <dc:StartTLSTrustCredential id="LDAPtoIdPCredential" xsi:type="sec:X509ResourceBacked">
        <sec:Certificate>%
          {idp.attribute.resolver.LDAP.trustCertificates}</sec:Certificate>
        </dc:StartTLSTrustCredential>
      -->
      </resolver:DataConnector>
      ```

      **UTILITY FOR OPENLDAP ADMINISTRATOR:**
        * ```ldapsearch -H ldap:// -x -b "dc=example,dc=it" -LLL dn```
          * the baseDN ==> ```ou=people, dc=example,dc=it``` (branch containing the registered users)
          * the bindDN ==> ```cn=admin,dc=example,dc=it``` (distinguished name for the user that can made queries on the LDAP)


10. Enrich IDP logs with the authentication error occurred on LDAP
  * ```vim /opt/shibboleth/conf/logback```

    ```xml
    <!-- Logs LDAP related messages -->
    <logger name="org.ldaptive" level="${idp.loglevel.ldap:-WARN}"/>
 
    <!-- Logs on LDAP user authentication -->
    <logger name="org.ldaptive.auth.Authenticator" level="INFO" />
    ```

11. Build the **attribute-resolver.xml** to define which attributes your IdP can release *(a Federation may distribute an attribute resolver compliant with its reccomendations, here we will give a basic configuration only)*:
  * ```vim /opt/shibboleth/conf/services.xml```

    ```xml
    <value>%{idp.home}/conf/attribute-resolver.xml</value>
 
    must become:
 
    <value>%{idp.home}/conf/attribute-resolver-full.xml</value>
    ```

  *  ```vim /opt/shibboleth-idp/conf/attribute-resolver-full.xml```
    * Remove comment from "**Schema: Core Schema attributes**"
    * Remove comment from "**Schema: InetOrgPerson attributes**"
    * Remove comment from "**Schema: eduPerson attributes**"

    (Obviously, this schemas are the default ones, but for new attributes, your LDAP could need some new schemas)
    
    * Remove the comment from the LDAP Data Connector configured previously on ```ldap.properties```

12. Translate the IdP messages in your language:
  * Get the files translated in your language from [Shibboleth page](https://wiki.shibboleth.net/confluence/display/IDP30/MessagesTranslation) for:
    * **login page** (authn-messages_it.properties)
    * **user consent/terms of use page** (consent-messages_it.properties)
    * **error pages** (error-messages_it.properties)
  
  * Put all the downloded files into ```/opt/shibboleth-idp/messages``` directory
    * Restart Tomcat by: ```service tomcat restart```

13. Enable the SAML2 support by changing the ```idp-metadata.xml``` and disabling the SAML v1.x deprecated support:
  * ```vim /opt/shibboleth-idp/metadata.xml```
    ```bash
    STRINGS TO REMOVE:
      – urn:oasis:names:tc:SAML:1.1:protocol
      – urn:mace:shibboleth:1.0
      – Entire endpoint with Binding="urn:oasis:names:tc:SAML:1.0:bindings:SOAPbinding" (and change the index in the right way)
      – <NameIDFormat>urn:mace:shibboleth:1.0:nameIdentifier</NameIDFormat>
      – Entire endpoint with Binding="urn:mace:shibboleth:1.0:profiles:AuthnRequest" (and change the index in the right way)
      – 8443 (everywhere, because we don't use it)
      
    IN THE ATTRIBUTE-AUTHORITY SECTION:
    – Replace "urn:oasis:names:tc:SAML:1.1:protocol" with "urn:oasis:names:tc:SAML:2.0:protocol"
    - Remove the comment from the AttributeService SAML2 and comment out the SAMLv1 one.
    ```

14. Register your IdP on your federation with the metadata found on:
  *  ```https://##idp.example.it##/idp/shibboleth```

15. Configure the IdP to retrieve the Federation Metadata:
  * ```cd /opt/shibboleth-idp/conf```
  * ```vim metadata-providers.xml```

    ```xml
    <MetadataProvider
          id="URLMD-Federation"
          xsi:type="FileBackedHTTPMetadataProvider"
          backingFile="%{idp.home}/federation-test-metadata-sha256.xml"
          metadataURL="https://www.exampleFed.it/metadata/federation-test-metadatasha256.xml">

          <!--
              Verify the signature on the root element of the metadata aggregate
              using a trusted metadata signing certificate.
          -->
          <MetadataFilter xsi:type="SignatureValidation" requireSignedRoot="true" certificateFile="${idp.home}/metadata/federation-cert.pem"/>
 
          <!--
              Require a validUntil XML attribute on the root element and make sure its value is no more than 14 days into the future. 
          -->
          <MetadataFilter xsi:type="RequiredValidUntil" maxValidityInterval="P14D"/>

          <!-- Consume all SP metadata in the aggregate -->
          <MetadataFilter xsi:type="EntityRoleWhiteList">
            <RetainedRole>md:SPSSODescriptor</RetainedRole>
          </MetadataFilter>
    </MetadataProvider>
    ```

  * Retrive the Federation Certificate used to verify its signed metadata:
    *  ```wget https://www.exampleFed.it/certificate/federation-cert.pem -O /opt/shibboleth-idp/metadata/federation-cert.pem```

  * Check the validity:
    *  ```cd /opt/shibboleth-idp/metadata```
    *  ```openssl x509 -in federation-cert.pem -fingerprint -sha1 -noout```
    *  ```openssl x509 -in federation-cert.pem -fingerprint -md5 -noout```
  
15. Reload service with id ```shibboleth.MetadataResolverService``` to retrieve the Federation Metadata:
  *  ```cd /opt/shibboleth-idp/bin```
  *  ```./reload-service.sh -id shibboleth.MetadataResolverService```

### Configure Attribute Filter for Research and Scholarship Entity Category

1. Retrieve the attribute filter Research and Scholarship compliant:
  *  Download the [R&S Attribute Filter](../master/attribute-filter-rs.xml) inside ```cd /opt/shibboleth-idp/conf```
  
2. Modify your ```services.xml```:
  * ```vim /opt/shibboleth-idp/conf/services.xml```

    ```xml
    <util:list id ="shibboleth.AttributeFilterResources">
        <value>%{idp.home}/conf/attribute-filter.xml</value>
        <value>%{idp.home}/conf/attribute-filter-rs.xml</value>
     </util:list>
     ```

3. Reload service with id ```shibboleth.AttributeFilterService``` to refresh the Attribute Filter followed by the IdP:
  *  ```cd /opt/shibboleth-idp/bin```
  *  ```./reload-service.sh -id shibboleth.AttributeFilterService```
