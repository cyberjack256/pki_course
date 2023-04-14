## Setting up a PKI Server
1. Install OpenSSL on a server that will act as the PKI server
2. Generate the CA private key file:
```bash
# Generate the CA private key file
openssl genrsa -out demo-ca.key 2048
```
3. Provision the CA by generating a self-signed root certificate
```bash
openssl req -x509 -new -nodes \
  -key demo-ca.key \
  -sha256 -subj "/C=US/ST=TX/L=Austin/O=LogsRLife/OU=IT/CN=demo-ca" \
  -days 365 -out demo-ca.crt # 1-year CA
```
* This command generates a root certificate that is valid for one year and is signed with the private key generated in step 2.

## Generateing PKI Certificates
### LogShipper Server

1. Create the CSR configuration file for the LogShipper server

```bash
cat > logshipper-server.csr.conf <<EOF
[ req ]
default_bits = 2048
prompt = no
default_md = sha256
req_extensions = req_ext
distinguished_name = dn

[ dn ]
C = US
ST = TX
L = Austin
O = LogsRLife
OU = IT
CN = logshipper-server

[ req_ext ]
subjectAltName = @alt_names

[ alt_names ]
DNS.1   = logshipper-server
IP.1    = 172.16.0.20
EOF

```
2. Generate the private key and CSR for the LogShipper server:

```bash
openssl req -new -newkey rsa:2048 -nodes -out logshipper-server.csr -keyout logshipper-server.key \
  -config logshipper-server.csr.conf
```

3. Sign the CSR with the CA to generate a certificate:
```bash
openssl x509 -req -in logshipper-server.csr -CA demo-ca.crt -CAkey demo-ca.key -CAcreateserial \
  -out logshipper-server.crt -days 90 -sha256
```

### LogScale Server

1. Create the CSR configuration file for the LogScale server:

```bash
cat > logscale-server.csr.conf <<EOF
[ req ]
default_bits = 2048
prompt = no
default_md = sha256
req_extensions = req_ext
distinguished_name = dn
x509_extensions = v3_req

[ dn ]
C = US
ST = TX
L = Austin
O = LogsRLife
OU = IT
CN = 172.16.0.10

[ req_ext ]
subjectAltName = @alt_names

[ v3_req ]
subjectAltName = @alt_names
extendedKeyUsage = serverAuth
keyUsage = keyEncipherment, dataEncipherment

[ alt_names ]
IP.1    = 172.16.0.10
URI.1    = https://172.16.0.10:8080
EOF
```

2. Generate the private key and CSR for the LogScale server:

```bash

openssl req -new -newkey rsa:2048 -nodes -out logscale-server.csr -keyout logscale-server.key \
  -config logscale-server.csr.conf
```

3. Sign the CSR with the CA to generate a certificate:

```bash
openssl x509 -req -in logscale-server.csr -CA demo-ca.crt -CAkey demo-ca.key -CAcreateserial \
  -out logscale-server.crt -days 90 -sha256 -extfile logscale-server.csr.conf -extensions v3_req
```

## Generating the Trust Store

1. Create a PKCS12 keystore containing the LogScale server's private key and certificate:
```bash
openssl pkcs12 -export -in logscale-server.crt -inkey logscale-server.key \
  -out keystore.p12 -name logscale-server \
  -passout pass:logsrlife
```
This command exports the LogScale server's private key and certificate to a PKCS12 keystore named keystore.p12. Replace your_keystore_password with a secure password of your choice.

2. Create a truststore containing the demo CA certificate:
```bash
keytool -importcert -file demo-ca.crt -alias demo-ca \
  -keystore truststore.p12 -storetype PKCS12 \
  -storepass logsrlife -noprompt
```

This command imports the demo CA certificate into a PKCS12 truststore named truststore.p12. Replace your_truststore_password with a secure password of your choice


### LogScale Configuration in humio.conf

####Notes:
-Xmx2g: Sets the maximum heap size for the JVM to 2 gigabytes. The heap size determines the amount of memory available for the JVM to allocate objects.
-Xms2g: Sets the initial heap size for the JVM to 2 gigabytes. This value determines the initial amount of memory the JVM allocates at startup.
-XX:+UseG1GC: Enables the G1 (Garbage-First) garbage collector, which is designed for low-latency, high-throughput applications. G1GC is a modern garbage collector that provides better performance compared to older JVM garbage collectors.
-Dconfig.file=/humio-akka-application.conf: Sets the Akka configuration file to use for LogScale. This is a Java system property that tells Akka where to look for the configuration file.

```
# LogScale (Humio) Configuration
HUMIO_OPTS=-Xmx2g -Xms2g -XX:+UseG1GC -Dconfig.file=/humio-akka-application.conf
HUMIO_HTTP_BIND=0.0.0.0
HUMIO_SOCKET_BIND=0.0.0.0
PUBLIC_URL=https://172.16.0.10:8080

# TLS Configuration
TLS_SERVER=true
TLS_PROTOCOLS=TLSv1.2,TLSv1.3
TLS_KEYSTORE_LOCATION=/etc/humio/keystore.p12
TLS_KEYSTORE_PASSWORD=your_keystore_password
TLS_KEYSTORE_TYPE=PKCS12
TLS_TRUSTSTORE_LOCATION=/etc/humio/truststore.p12
TLS_TRUSTSTORE_PASSWORD=your_truststore_password
TLS_TRUSTSTORE_TYPE=PKCS12
```

## LogScale Akka Configuration
The Akka configuration file, humio-akka-application.conf, is written in the HOCON (Human-Optimized Config Object Notation) format. HOCON is a human-readable data format primarily used for configuration files in the Scala and Java ecosystems. It's a superset of JSON and is designed to be more compact and less verbose than JSON, XML, or other configuration formats. HOCON is used by Akka, Play Framework, and other Lightbend projects.

1. Create a new file called humio-akka-application.conf with the following content:

```bash
cat > humio-akka-application.conf << EOF
include "application"

akka {
  loglevel = "INFO"
  log-config-on-start = "on"

  http {
    server {
      ssl-config {
        keyManager = {
          stores = [
            { type = "PKCS12",
              path = "/etc/humio/certs/keystore.p12",
              password = "logsrlife"
            }
          ]
        }
        trustManager = {
          stores = [
            { type = "PKCS12",
              path = "/etc/humio/certs/truststore.p12",
              password = "logsrlife"
            }
          ]
        }
        protocol = "TLSv1.2"
      }
    }
  }
}
EOF
```

1. Create a directory on your host system to store the TLS certificates and keys, for example, /home/ec2-user/certs.

2. Copy the logscale-server.crt, logscale-server.key, demo-ca.crt, keystore.p12, and truststore.p12 files into the /home/ec2-user/certs directory on your host system.

3. Run the Docker container with the updated volume mapping for the TLS certificates:

```bash
sudo docker run -d --restart=always \
  -v /opt/data:/data \
  -v /opt/data/kafka-data:/kafka-data \
  -v /home/ec2-user/certs:/etc/humio/certs:ro \
  -v /home/ec2-user/humio.conf:/etc/humio/humio.conf:ro \
  -v /home/ec2-user/humio-akka-application.conf:/humio-akka-application.conf:ro \
  --add-host ip-172-16-0-10.ec2.internal:127.0.0.1 \
  -p 8080:8080 \
  --name=logscale \
  --ulimit="nofile=8192:8192" \
  humio/humio:stable
```

This Docker run command maps the /home/ec2/certs directory from your host system to /etc/humio/certs in the Docker container, which makes the TLS certificates and keys available to the LogScale server. The humio.conf file is also mapped from your host system to the container.

#### Now, your LogScale server will be configured with the specified TLS settings, and the certificates will be available to the container.

## Troubleshooting
1. Verify the validity of the CA certificate:
```bash
openssl x509 -in demo-ca.crt -noout -text
```
This command will display the details of the CA certificate. Verify that the certificate's expiry date, subject, and issuer match your expectations.

2. Verify the validity of the LogShipper server certificate:
```bash
openssl x509 -in logshipper-server.crt -noout -text
```
This command will display the details of the LogShipper server certificate. Verify that the certificate's expiry date, subject, and issuer match your expectations.

3. Verify the validity of the LogScale server certificate:
```bash
openssl x509 -in logscale-server.crt -noout -text
```
This command will display the details of the LogScale server certificate. Verify that the certificate's expiry date, subject, and issuer match your expectations.

4. Verify the certificate chain:
```bash
openssl verify -CAfile demo-ca.crt logshipper-server.crt
```
This command will verify the LogShipper server certificate against the CA certificate. Verify that the output shows "OK."

```bash
openssl verify -CAfile demo-ca.crt logscale-server.crt
```
This command will verify the LogScale server certificate against the CA certificate. Verify that the output shows "OK."

5. Verify the private keys:
```bash
openssl rsa -in demo-ca.key -check
```
This command will verify the CA private key.

```bash
openssl rsa -in logshipper-server.key -check
```
This command will verify the LogShipper server private key.

```bash
openssl rsa -in logscale-server.key -check
```
This command will verify the LogScale server private key.

### curl(35) Error
If you are receiving a curl error (35) from the LogScale Server when executing
```
curl https://127.0.0.1:8080
```
This indicates that the SSL/TLS handshake between the curl client and the server failed due to a problem with the SSL/TLS certificate or configuration. This may be due to several reasons, such as an invalid or expired SSL/TLS certificate, incorrect configuration of the SSL/TLS settings, or mismatch between the hostname in the certificate and the hostname used to connect to the server.

To troubleshoot the issue, you can try the following steps:

1. Check the certificate details: Verify that the certificate presented by the server is valid and matches the expected details such as the hostname, issuer, and expiration date. You can use the openssl command to view the certificate details:
```bahs
openssl s_client -showcerts -connect 127.0.0.1:8080
```
This command will connect to the LogScale server on port 8080 and display the certificate details.

2. Check the SSL/TLS configuration: Verify that the SSL/TLS configuration on the server matches the SSL/TLS configuration used by the client. Check that the SSL/TLS protocol version, cipher suite, and other settings are correct.

3. Check the hostname: Verify that the hostname used to connect to the server matches the hostname in the SSL/TLS certificate. If there is a mismatch, you can update the hostname in the certificate or use the correct hostname to connect to the server.

4. Check the firewall settings: Verify that the firewall settings allow traffic to pass through the required ports and protocols.

5. Check the curl settings: Try using the `-k` or `--insecure` option with the curl command to skip the SSL/TLS certificate verification. If the command succeeds with this option, it indicates that there is an issue with the SSL/TLS certificate or configuration.

## Docker Commands

1. Restart container with new settings
```bash
sudo docker restart $(sudo docker ps -a | grep 'logscale' | awk '{print $1}')

```
