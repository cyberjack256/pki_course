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
IP.1    = 172.16.0.10
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

## Generating PKI Certificates
1. Generate a private key and a Certificate Signing Request (CSR) for the service that requires a certificate. For example, to generate a CSR for a web server:
```bash
openssl req -new -newkey rsa:2048 -nodes -out logsrlife-logscale-server.csr -keyout logsrlife-logscale-server.key \
  -config logsrlife-logscale-server.csr.conf
```
* This command generates a private key and a CSR based on the configuration file logsrlife-logscale-server.csr.conf. Modify the configuration file to match your organization's details and the service that requires a certificate.

2. Sign the CSR with the CA to generate a certificate:

```bash
openssl x509 -req -in logsrlife-logscale-server.csr -CA demo-ca.crt -CAkey demo-ca.key -CAcreateserial \
  -out logsrlife-logscale-server.crt -days 90 -sha256
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

[ dn ]
C = US
ST = TX
L = Austin
O = LogsRLife
OU = IT
CN = logscale-server.logsrlife.example

[ req_ext ]
subjectAltName = @alt_names

[ alt_names ]
DNS.1   = logscale-server.logsrlife.example
DNS.2   = logscale-server
IP.1    = 172.16.0.20:8080
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
  -out logscale-server.crt -days 90 -sha256
```

## Generating the Trust Store

1. Create a PKCS12 keystore containing the LogScale server's private key and certificate:
```bash
openssl pkcs12 -export -in logscale-server.crt -inkey logscale-server.key \
  -out keystore.p12 -name logscale-server \
  -passout pass:your_keystore_password
```
This command exports the LogScale server's private key and certificate to a PKCS12 keystore named keystore.p12. Replace your_keystore_password with a secure password of your choice.

2. Create a truststore containing the demo CA certificate:
```bash
keytool -importcert -file demo-ca.crt -alias demo-ca \
  -keystore truststore.p12 -storetype PKCS12 \
  -storepass your_truststore_password -noprompt
```

This command imports the demo CA certificate into a PKCS12 truststore named truststore.p12. Replace your_truststore_password with a secure password of your choice.

With the keystore and truststore files created, you can now use them in your LogScale server's configuration to enable secure communication using TLS. Remember to protect the keystore and truststore files, as they contain sensitive information.

### LogScale Configuration in humio.conf

```
# LogScale (Humio) Configuration
HUMIO_HTTP_BIND=0.0.0.0
HUMIO_SOCKET_BIND=0.0.0.0
PUBLIC_URL=http://172.16.0.20:8080

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

1. Create a directory on your host system to store the TLS certificates and keys, for example, /home/ec2-user/certs.

2. Copy the logscale-server.crt, logscale-server.key, demo-ca.crt, keystore.p12, and truststore.p12 files into the /home/ec2-user/certs directory on your host system.

3. Run the Docker container with the updated volume mapping for the TLS certificates:

```bash
sudo docker run -d --restart=always \
  -v /opt/data:/data \
  -v /opt/data/kafka-data:/kafka-data \
  -v /home/ec2-user/certs:/etc/humio/certs:ro \
  -v /home/ec2-user/humio.conf:/etc/humio/humio.conf:ro \
  --add-host ip-172-16-0-20.ec2.internal:127.0.0.1 \
  --net=host \
  --name=logscale \
  --ulimit="nofile=8192:8192" \
  humio/humio:stable
```

This Docker run command maps the /home/ec2/certs directory from your host system to /etc/humio/certs in the Docker container, which makes the TLS certificates and keys available to the LogScale server. The humio.conf file is also mapped from your host system to the container.

#### Now, your LogScale server will be configured with the specified TLS settings, and the certificates will be available to the container.

## Troubleshooting

openssl x509 -in /etc/ec2-user/logscale-server.crt -noout -text

