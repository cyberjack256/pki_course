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

## Generating PKI Certificates
1. Generate a private key and a Certificate Signing Request (CSR) for the service that requires a certificate. For example, to generate a CSR for a web server:
```bash
openssl req -new -newkey rsa:2048 -nodes -out logsrlife-nginx.csr -keyout logsrlife-nginx.key \
  -config logsrlife-ldap.csr.conf
```
* This command generates a private key and a CSR based on the configuration file logsrlife-ldap.csr.conf. Modify the configuration file to match your organization's details and the service that requires a certificate.

2. Sign the CSR with the CA to generate a certificate:

```bash
openssl x509 -req -in logsrlife-nginx.csr -CA demo-ca.crt -CAkey demo-ca.key -CAcreateserial \
  -out logsrlife-nginx.crt -days 90 -sha256
```
* This command signs the CSR with the CA and generates a certificate that is valid for 90 days.

* Replace logsrlife-nginx with the appropriate name of the service for which you are generating a certificate.

You can then use the generated certificate and private key in the appropriate service configuration. Repeat steps 1 and 2 to generate certificates for additional services as needed.

Note: Make sure to keep the CA private key (demo-ca.key) secure and protected. Anyone with access to the CA private key can generate valid certificates for any service, so it should be kept in a safe location and only accessible by authorized personnel.

## Install NGINX as a proxy server

1. Install Nginx

```bash
sudo yum install nginx.
```

2. Start Nginx

```bash
sudo systemctl start nginx.
```

3. Check that Nginx is running 

```bash
sudo systemctl status nginx
```

**Note**: The output should indicate that the service is active (running).

4. Generate server certificate using PKI Server:

```bash
openssl req -new -key logsrlife-nginx.key -out logsrlife-nginx.csr -config logsrlife-nginx.csr.conf
openssl x509 -req -in logsrlife-nginx.csr -CA demo-ca.crt -CAkey demo-ca.key -CAcreateserial -out logsrlife-nginx.crt -days 90
sudo cp logsrlife-nginx.crt /etc/ssl/certs/
sudo cp logsrlife-nginx.key /etc/ssl/private/
```

4. Configure Nginx to act as a reverse proxy for LogScale. This can be done by creating a new server block in the Nginx configuration file /etc/nginx/nginx.conf or by creating a new configuration file in the /etc/nginx/conf.d directory.

```config
server {
    listen 443 ssl;
    server_name logscale-server.logsrlife.local;
    ssl_certificate /etc/ssl/certs/logsrlife-nginx.crt;
    ssl_certificate_key /etc/ssl/private/logsrlife-nginx.key;

    location / {
        proxy_pass http://172.20.16.XX:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```

5. Restart Nginx service to apply the changes.
```bash
sudo systemctl restart nginx
```

6. Check that the Nginx service is running without any errors.

```bash
sudo systemctl status nginx
```

7. You can check the SSL certificate of the server configured in NGINX by running the following command:

```bash
openssl s_client -connect logscale-server.logsrlife.local:443
```
* This command initiate an SSL connection to the server and return information about the SSL certificate, including the subject, issuer, and expiration date. If the certificate is valid and properly installed, the output should indicate that the certificate is trusted and the connection is established without any errors.


## Configure LogScale to use the reverse proxy

1. Generate a certificate for the LogScale server using the following command:

```bash
openssl req -new -newkey rsa:2048 -nodes -out logsrlife-logscale.local.csr -keyout logsrlife-logscale.local.key -config logsrlife-logscale.local.csr.conf
```

2. Sign the certificate using the PKI CA created earlier. This can be done using the following command:

```bash
openssl x509 -req -in logsrlife-logscale.local.csr -CA demo-ca.crt -CAkey demo-ca.key -CAcreateserial -out logsrlife-logscale.local.crt -days 90 -sha256 -extfile logsrlife-logscale.local.csr.conf -extensions v3_req
```

3. Combine the signed certificate and private key into a single file (bundle) using the following command:

```bash
cat logsrlife-logscale.local.crt logsrlife-logscale.local.key > logsrlife-logscale.local.pem
```
4. Copy the logsrlife-logscale.local.pem file to the LogScale server.

5. Configure LogScale to use NGINX as a reverse proxy by modifying the /opt/humio-server/conf/server.conf file.

```conf
upstream humio-backends {
  zone humio 32000000;
  server logsrlife-logscale.local:8080 max_fails=0 fail_timeout=10s max_conns=256;
}

server {
  listen 80;
  server_name logsrlife-logscale.local;
  return 301 https://$server_name$request_uri;
}

server {
  listen 443 ssl;
  server_name logsrlife-logscale.local;

  ssl_certificate /path/to/logsrlife-logscale.local.pem;
  ssl_certificate_key /path/to/logsrlife-logscale.local.key;

  location /internal/humio {
    proxy_set_header X-Forwarded-Prefix /internal/humio;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Real-IP $remote_addr;

    proxy_redirect http:// https://;
    expires off;

    # Following headers are important to establish a WebSocket connection to LSP according to docs as of 3/22/23.
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
    proxy_http_version 1.1;
    proxy_set_header Host $host;

    proxy_pass http://humio-backends;
  }
}

```
6. Restart the LogScale server for the changes to take effect.

```bash
sudo systemctl restart humio-server
```

7. Now your LogScale server should be configured to use NGINX as a reverse proxy.

## Generate the CSR file for the LogShipper LogScale and  Server and Certs for LDAP


1. Create the CSR file for each of the servers. This will aid in replicating the settings for our LogShipper, LDAP, and LogScale servers.

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

[v3_req]
subjectAltName = @alt_names
extendedKeyUsage = serverAuth
keyUsage = keyEncipherment, dataEncipherment

[ alt_names ]
DNS.1   = logshipper-server
DNS.2   = logshipper-server.logsrlife.example
# Get new local IP from admin
IP.1    = X.X.X.X
EOF
```

```bash
cat > ldap-server.csr.conf <<EOF
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
CN = ldap-server

[ req_ext ]
subjectAltName = @alt_names

[v3_req]
subjectAltName = @alt_names
extendedKeyUsage = serverAuth
keyUsage = keyEncipherment, dataEncipherment

[ alt_names ]
DNS.1   = ldap-server
DNS.2   = ldap-server.logsrlife.example
# Get new local IP from admin
IP.1    = X.X.X.X
EOF
```

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

[v3_req]
subjectAltName = @alt_names
extendedKeyUsage = serverAuth
keyUsage = keyEncipherment, dataEncipherment

[ alt_names ]
DNS.1   = logscale-server.logsrlife.example
DNS.2   = logscale-server
# Get new local IP from admin
IP.1    = X.X.X.X
EOF
```


1. Generate a private key for the LDAP server

```bash
openssl genrsa -out logsrlife.key 2048

```

2. Create a Certificate Signing Request (CSR) for our LDAP server:

Note: 
* The CSR configuration specifies a 2048-bit key, which is considered sufficient for most purposes.
* The distinguished name (DN) includes standard fields such as country (C), state (ST), locality (L), organization (O), organizational unit (OU), and common name (CN). These fields help identify the LDAP server and its owner.
* The subject alternative name (SAN) extension allows the certificate to include additional identifiers besides the common name (CN). In this case, the SAN includes two DNS names (ldap-server and ldap-server.logsrlife.example) and one IP address (X.X.X.X). The DNS names may be useful for clients that use DNS resolution to find the LDAP server, while the IP address may be useful for clients that use IP addresses directly.
* The extended key usage (EKU) extension specifies the purposes for which the certificate can be used. * The serverAuth value indicates that the certificate can be used for server authentication, which is appropriate for an LDAP server. The keyUsage extension specifies the cryptographic operations that the certificate can be used for. In this case, the keyEncipherment and dataEncipherment values indicate that the certificate can be used for encrypting and decrypting data.

```bash
openssl req -new -key logsrlife.key -out logsrlife.csr \
  -config logsrlife-ldap.csr.conf
```

1. Copy the CA certificate to the appropriate location on the LDAP Server

```bash
sudo cp demo-ca.crt /etc/ssl/certs/logsrlife-ca.crt
```

4. Copy the server certificate to the appropriate location on the LDAP Server

```bash
sudo cp logsrlife.crt /etc/ssl/certs/logsrlife.crt
sudo cp logsrlife.key /etc/ssl/private/logsrlife.key
```


## Install OpenLDAP on the LDAP Server

1. Install openldap on the LDAP Server

```bash
sudo yum install openldap-servers openldap-clients
```


2. After installation of OpenLDAP, there is no default root DN or password set up. We will need to create a root DN and password.

To create a new root DN and password, we will use the slappasswd utility to generate a hashed password. The next step is to then create an LDIF file with the new root DN and hashed password:

3. Generate an SSHA hash for the rootdn password logallthethings
```bash
slappasswd -s logallthethings -h {SSHA}
```

4. Create a new LDIF file named rootdn.ldif. Replace xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx with the hashed password of our LDAP administrator:

```bash
sudo nano rootdn.ldif
```

```text
dn: cn=admin,dc=logsrlife,dc=local
objectClass: simpleSecurityObject
objectClass: organizationalRole
cn: admin
userPassword: {SSHA}xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx

```

5. Use the ldapadd utility to add the new root DN and password to the OpenLDAP database:

```bash
ldapadd -x -D "cn=admin,dc=logsrlife,dc=local" -w <our-password> -f rootdn.ldif
```

6. Edit the `slapd.conf` file 
```bash
sudo nano /etc/openldap/slapd.conf
```

**Note**: For an actual environment we would likely want to make the following amendments to the config:

suffix: Replace dc=logsrlife,dc=local with our own domain name components separated by ,dc=.
rootdn: Replace admin,dc=logsrlife,dc=local with the DN of our LDAP administrator account.
rootpw: Replace xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx with the hashed password of our LDAP administrator account.
directory: Replace /var/lib/ldap with the path to the directory where we want to store our LDAP database.
TLSCACertificateFile: Replace /etc/openldap/certs/ca-cert.pem with the path to our CA certificate file.
TLSCertificateFile: Replace /etc/openldap/certs/server-cert.pem with the path to our server certificate file.
TLSCertificateKeyFile: Replace /etc/openldap/certs/server-key.pem with the path to our server private key file.



Note: Indexes in OpenLDAP are used to speed up searches by pre-sorting the data. The index directive in slapd.conf is used to specify which attributes should be indexed.

Note: There are four attributes that are indexed: objectClass, cn, sn, and uid. The eq keyword means that exact matching is used for the attribute, while pres and sub refer to present and substring matching respectively.

```config

# Define global options
include         /etc/openldap/schema/core.schema
pidfile         /var/run/openldap/slapd.pid
argsfile        /var/run/openldap/slapd.args

# Define database
database        bdb
suffix          "dc=logsrlife,dc=example"
rootdn          "cn=admin,dc=logsrlife,dc=example"
rootpw          {SSHA}xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
directory       /var/lib/ldap

# Define indexes
index objectClass eq
index cn pres,sub,eq
index sn pres,sub,eq
index uid pres,sub,eq

# Define access controls
# Allow the admin user full access to the LDAP directory
# Allow all authenticated users to search the LDAP directory
# Allow users to modify their own entries
# Deny anonymous access
# Require secure connections for sensitive data
access to *
  by dn.exact="cn=admin,dc=logsrlife,dc=example" write 
  by users read
  by self write
  by anonymous auth
  by * none
  by ssf=128 self write

# Enable SSL/TLS
# Enable SSL/TLS
TLSCACertificateFile /path/to/demo-ca.crt
TLSCertificateFile /path/to/logsrlife-ldap.crt
TLSCertificateKeyFile /path/to/logsrlife-ldap.key
```

```bash
# Start and enable the service
sudo systemctl start slapd
sudo systemctl enable slapd
```


```bash
# Add Group to LDAP
sudo ldapadd -x -D "cn=admin,dc=logsrlife,dc=local" -w admin_password <<EOF
dn: cn=security_operations,ou=Groups,dc=logsrlife,dc=local
objectClass: groupOfNames
cn: security_operations
member: uid=charlotte,ou=staff,dc=logsrlife,dc=local
EOF

```
We'll use the same hashed password for our analyst user for ease of lab

```bash
# Add User to LDAP
sudo ldapadd -x -D "cn=admin,dc=logsrlife,dc=local" -w admin_password <<EOF
dn: uid=charlotte,ou=staff,dc=logsrlife,dc=local
objectClass: inetOrgPerson
objectClass: posixAccount
uid: charlotte
cn: Charlotte
sn: Doe
userPassword: {SSHA}xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
uidNumber: 10001
gidNumber: 10001
homeDirectory: /home/charlotte
loginShell: /bin/bash
EOF

```

## Configure LogScale for LDAP Authentication

1. Make sure that a root account exists on the LogScale server. You can either add the user through the User Administration Page in the Web UI, or use the API to add the root user.
2. Create a file named humio-config.env in the /etc/humio directory.
3. Add the following variables to the humio-config.env file:

```config
AUTHENTICATION_METHOD=ldap
LDAP_AUTH_PROVIDER_URL=ldap://logsrlife-ldap.local:389
LDAP_AUTH_PRINCIPAL=cn=charlotte,ou=staff,dc=logsrlife,dc=local
LDAP_DOMAIN_NAME=logsrlife.local
AUTO_CREATE_USER_ON_SUCCESSFUL_LOGIN=true
```

NOTE:
* AUTHENTICATION_METHOD: Set to ldap to enable LDAP authentication.
* LDAP_AUTH_PROVIDER_URL: The URL of your LDAP server.
* LDAP_AUTH_PRINCIPAL: The principal used to bind to the LDAP server. In this case, it's the username and location of Charlotte in the LDAP directory.
* LDAP_DOMAIN_NAME: The domain name used by your LDAP server.
* AUTO_CREATE_USER_ON_SUCCESSFUL_LOGIN: When set to true, it will create new users in LogScale on successful login if that user doesn't already exist.

4. Restart the LogScale server to apply the changes.
Test the LDAP authentication by logging in with Charlotte's credentials.
