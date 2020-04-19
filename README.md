# RabbitMQ with SSL Configuration and Management in Docker

> RabbitMQ and SSL made easy for tests.


This repository aims at building a RabbitMQ container with SSL enabled.  
Generation of the server certificates, as well as server configuration, are performed during
the image's build. A client certificate is generated when a container is created from this image.

It is recommended to mount a volume so that the client certificate can be reached from the
host system. Client certificates are generated under the **/home/client** directory.


## To build this image

```
cd tests && ./build.sh
```

The generated image doesn't contains SSL certificates for the server side, use Dockerfile.AutoSSL instead.

```
docker build - -t rabbitmq-with-ssl:latest < Docker.AutoSSL
```

## To run this image

```
mkdir -p /tmp/docker-test \
	&& rm -rf /tmp/docker-test/* \
	&& docker run -d --rm -p 5671:5671 -p 8080:15671 -v /tmp/docker-test:/home/client rabbitmq-with-ssl:latest
```

Here, we bind the port 5671 from the container on the 5671 port on the local host and port for the management interface 15671 to 8080 port on the local host.  
We also share a local directory with the container, to retrieve the client certificate.
You can verify client certificates were generated with `ls /tmp/docker-test`. This directory contains
a key store and a trust store, both in the PKCS12 format.
Management interface is available at https://localhost:8080


## To stop the container

`docker stop <container-id>` will stop the container.  
If you kept the `--rm` option, it will be deleted directly.


## To run quick tests

```
cd tests && ./test.sh
```

Download all the files (zip) and extract it to make sure it contains
certificate.crt : Certificate generated for your domain name.
private.key: Private key of your certificate.
ca_bundle.crt: Intermediate CA (Certificate Authority)
1.2 Generate self signed certificate
Another way to get SSL certificate is to generate self-signed certificate using keytool and the rest of the steps are same.


2.1 Conversion to PKCS-12
Please note that we need to add certificate as well as private key to the keystore which is not possible in JKS format, hence certificate and private key need to be converted into .p12 format
<pre><code>
# openssl pkcs12 -export -in zip/certificate.crt -inkey zip/private.key -name expertwall -out zip/expertwall-free-PKCS-12.p12
</code></pre>
The assumption here is that zip folder is present in the current directory of `openssl` and contains all the required files (certificate.crt, private.key, and ca_bundle.crt). Above command will generate zip/expertwall-free-PKCS-12.p12


2.2 Import PKCS-12 file format in JKS keystore
JDK would be require for this as bin folder contains a utility called keytool

<pre><code>
# keytool -importkeystore -deststorepass myPassPhrase -destkeystore zip/expertwall-keystore.jks -srckeystore zip/expertwall-free-PKCS-12.p12 -srcstoretype PKCS12
</code></pre>
Above command will generate free/expertwall-keystore.jks file with imported certificate and private key.

2.3 Import intermediate trust certificate
In order to verify your-trusted-certificate an intermediate trust certificate is also required. Following command will import ca bundle (Certificate authority) within the same keystore.

<pre><code>
# keytool -import -alias bundle -trustcacerts -file free/ca_bundle.crt -keystore free/expertwall-keystore.jks
</code></pre>

volla! your keystore is ready (contains, your SSL certificate, your private key, and certificate of CA)

Configure Spring Boot application
Open application.yml file and add following code, specify the port 8443 to run SSL port and also enter the details of keystore.


<pre><code>
server:
  contextPath: /expertwall
  port: 8443
  ssl:
    key-store: classpath:expertwall-keystore.jks
    key-store-password: myPassPhrase
    key-alias: expertwall

</code></pre>




