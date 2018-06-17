## Enable SSL on Hadoop services(HDFS/YARN/MR)


### Create certificates and Truststore/Keystore

**Create certificate authority :** 

**Step 1**: Install OpenSSL, for example on CentOS run:

    #yum install openssl

**Step 2**: Generate a CA signing key and certificate:

    #openssl genrsa -out ca.key 8192
    #openssl req -new -x509 -extensions v3_ca -key ca.key -out ca.crt -days 365

(Create CA with Common Name(CN) set with name '*Root CA*')

**Step 3**: Set up the CA directory structure and copy CA key and CA crt created in step 2 to /root/CA/private and /root/CA/certs respectively:

    #mkdir -p -m 0700 /root/CA/{certs,crl,newcerts,private}
    #mv ca.key /root/CA/private;mv ca.crt /root/CA/certs

**Step 4**: Add required files and set permissions on the ca.key:

    #touch /root/CA/index.txt; echo 1000 > /root/CA/serial
    #chmod 0400 /root/CA/private/ca.key

**Step 5**: Configure openssl.cnf to set the directory path to /root/CA/:

    #vi /etc/pki/tls/openssl.cnf
    [...]
    [ CA_default ]
    dir             = /root/CA/         # Where everything is kept
    [...]

### Create and Sign CSR :

**Step 6**: Create a directory which will be used for new csr and certs: 

    #mkdir /var/tmp/SSL; cd /var/tmp/SSL

**Step 7**: Generate keys and csr using that corresponding key of each host. Make sure the common name portion of the certificate matches the hostname where the certificate will be deployed.

    #openssl genrsa -out <Hostname>.key 2048
    #openssl req -new -sha256 -key <Hostname>.key  -out <Hostname>.csr
    #openssl req -in <Hostname>.csr -noout -text

(Repeat the Step 7 for all the hosts which require cert)

 
**Step 8**:  Sign the all csr created in Step 7 using the CA created in 'Step 2': 

    #openssl x509 -req -CA /root/CA/certs/ca.crt -CAkey /root/CA/private/ca.key -in <.csr file> -out <Hostname>.crt -days 365 -CAcreateserial
    #openssl x509 -in <Hostname>.crt -noout -text

### Create jks keystore and truststore:

**Step 9**: Create PKCS12 keystore and convert it to JKS(Repeat this step for all the .key and .crt of each hosts): 

    #openssl pkcs12 -export -inkey <hostname>.key -in <hostname>.crt -certfile /root/CA/certs/ca.crt -out <hostname>.pfx
    #keytool -list -keystore <hostname>.pfx -storetype PKCS12 -v

**Step 10**: Convert the PKCS12 format to JKS: 

    #keytool -v -importkeystore -srckeystore <hostname>.pfx -srcstoretype PKCS12 -destkeystore <hostname>.jks -deststoretype JKS -srcalias 1 -destalias <hostname>

**Step 11**: Create a common truststore (as we have signed with CA cert we only need the CA cert in truststore): 

    #keytool -import -keystore truststore.jks -alias rootca -file ca.crt
    #cp truststore.jks all.jks 

### Distribute keystore and truststore to all hosts:

**Step 12**: On all the hosts in the cluster create the directory structure which will be used in configuration: 

    #mkdir -m 0755 /etc/security/{serverKeys,clientKeys}

**Step 13**: scp host specific jks ,truststore.jks and all.jks to the hosts: 

**Step 14**: Copy the jks files in the respective locations (Repeat this step on all hosts): 

      #cp all.jks /etc/security/clientKeys/
      #cp <hostname>.jks /etc/security/clientKeys/keystore.jks
      #chmod 440 /etc/security/clientKeys/keystore.jks
      #cp truststore.jks /etc/security/clientKeys/truststore.jks
      #chmod 444 /etc/security/clientKeys/truststore.jks

Copy server files :

    #cp <hostname>.jks /etc/security/serverKeys/keystore.jks
    #cp truststore.jks /etc/security/serverKeys/truststore.jks
    #chmod 440 /etc/security/serverKeys/*
    #chown -R yarn:hadoop /etc/security/{serverKeys,clientKeys}

### Configure services for SSL using keystores and truststore created in previous section:

**Step 15**: Configuring SSL for HDFS/YARN and MR

-From Ambari Set below properties in referred config files(From Filter search for property if no results then you can add property under custom <>-site section)
*core-site.xml*

    hadoop.rpc.protection=privacy
    hadoop.ssl.require.client.cert=false
    hadoop.ssl.hostname.verifier=DEFAULT
    hadoop.ssl.keystores.factory.class=org.apache.hadoop.security.ssl.FileBasedKeyStoresFactory
    hadoop.ssl.server.conf=ssl-server.xml
    hadoop.ssl.client.conf=ssl-client.xml

-And in *ssl-server.xml* configure keystore location and password

    ssl.server.keystore.keypassword: <keystore Key password>
    ssl.server.keystore.password: <Keystore Password>
    ssl.server.keystore.location: /etc/security/serverKeys/keystore.jks
    ssl.server.truststore.location :/etc/security/serverKeys/truststore.jks
    ssl.server.truststore.password: <truststore Password>

-In *ssl-client.xml* config:

    ssl.client.keystore.location: /etc/security/clientKeys/keystore.jks
    ssl.client.keystore.password: <keystorePassword>
    ssl.client.truststore.location: /etc/security/clientKeys/all.jks
    ssl.client.truststore.password: <TrustStorePassword>

*hdfs-site.xml*

    dfs.encrypt.data.transfer=true
    dfs.encrypt.data.transfer.algorithm=3des
    dfs.http.policy=HTTPS_ONLY
    dfs.datanode.https.address=<hostname>:50475
    dfs.namenode.https-address=<hostname>:50470

*mapred-site.xml*

    mapreduce.jobhistory.http.policy=HTTPS_ONLY
    mapreduce.jobhistory.webapp.https.address=0.0.0.0:19890

*yarn-site.xml*

    yarn.http.policy=HTTPS_ONLY
    yarn.log.server.url=https://<mapred-history-server-host>:19890/jobhistory/logs
    yarn.resourcemanager.webapp.https.address=<RM>:8090
    yarn.nodemanager.webapp.https.address=0.0.0.0:8044

**Step 16**: Restart all the required services from Ambari

### Configure Ambari truststore:

**Step 17**: Copy truststore.jks to ambari server to create ambari server truststore.

**Step 18**: Setup truststore for ambari server using the truststore copied in *Step 17*:  

    #ambari-server setup-security (option 4)

**Step 19**: Restart Ambari server

    #ambari-server restart
    
**Step 20**: Verify if Ambari  shows metrics in dashboard and also verify if quick links for hdfs/yarn/mr are accessible on https: 
