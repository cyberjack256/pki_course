## Create a Private CA to use in the Demo
```bash
# Generate the CA private key file
openssl genrsa -out demo-ca.key 2048

# Provision the CA
openssl req -x509 -new -nodes \
  -key demo-ca.key \
  -sha256 -subj "/C=US/ST=TX/L=Austin/O=LogsRLife/OU=IT/CN=demo-ca" \
  -days 365 -out demo-ca.crt # 1-year CA

# Example of how to Generate the server keys and CSRs replace the name.csr, name.key and name.csr.conf with the server creating the key for.
openssl req -new -newkey rsa:2048 -nodes -out logsrlife-nginx.csr -keyout logsrlife-nginx.key \
  -config logsrlife-ldap.csr.conf

# Sign the server CSR with the CA
openssl x509 -req -in logsrlife-nginx.csr -CA demo-ca.crt -CAkey demo-ca.key -CAcreateserial \
  -out logsrlife-nginx.crt -days 90 -sha256

```

Don't forget to add the CA root to the trust store on each server in the lab

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

4. Configure Nginx to act as a reverse proxy for LogScale. This can be done by creating a new server block in the Nginx configuration file `/etc/nginx/nginx.conf` or by creating a new configuration file in the `/etc/nginx/conf.d` directory.

```config
server {
    listen 443 ssl;
    server_name logscale-server.logsrlife.example;
    ssl_certificate /path/to/wer/certificate.pem;
    ssl_certificate_key /path/to/wer/key.pem;

    location / {
        proxy_pass http:/172.20.16.XX:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```

5. Configure LogScale to use NGINX as a reverse proxy

  * **Note**:
  In this configuration, the upstream block defines the LogScale server as a backend, and the server blocks define NGINX as a reverse proxy for LogScale.
  The first server block listens on port 80 and redirects all incoming requests to HTTPS. The second server block listens on port 443 and serves HTTPS requests, and contains the location block that proxies requests to the LogScale server.
  Within the location block, we set the necessary headers for establishing a WebSocket connection for the Language Server Protocol. We also set the proxy_pass directive to the humio-backends upstream, which contains the IP address of the LogScale server.
  Note that we will need to replace logscale_server_ip_address with the actual IP address of our LogScale server, and /path/to/ssl_certificate and /path/to/ssl_certificate_key with the file paths of our SSL certificate and key, respectively.
  * Creating the Bundle for crt: 
  ```bash
  cat /path/to/your/ssl_certificate.pem /path/to/your/root_certificates.pem > /path/to/your/public_key_fullchain.pem

  ```

```t
upstream humio-backends {
  zone humio 32000000;
  server logscale_server_ip_address:8080 max_fails=0 fail_timeout=10s max_conns=256;
}

server {
  listen 80;
  server_name example.com;
  return 301 https://$server_name$request_uri;
}

server {
  listen 443 ssl;
  server_name example.com;

  ssl_certificate /path/to/ssl_certificate;
  ssl_certificate_key /path/to/ssl_certificate_key;

  location /internal/humio {
    proxy_set_header X-Forwarded-Prefix /internal/humio;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Real-IP $remote_addr;

    proxy_redirect http:// https://;
    expires off;

    # Following headers are important to establish a WebSocket connection to LSP.
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
    proxy_http_version 1.1;
    proxy_set_header Host $host;

    proxy_pass http://humio-backends;
  }
}
```


## Generate the CSR file for the LogShipper LogScale and LDAP Server and Certs for LDAP


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


## Install Open LDAP on the LDAP Server

1. Install openldap on the LDAP Server

```bash
sudo yum install openldap-servers openldap-clients
```


2. After install of OpenLDAP, there is no default root DN or password set up. We will need to create a root DN and password.

    To create a new root DN and password, we will use the `slappasswd` utility to generate a hashed password. The next step is to then create an LDIF file with the new root DN and hashed password:

3. Generate an SSHA hash for the rootdn password `logallthethings`
```bash
slappasswd -s logallthethings -h {SSHA}
```

4. Create a new LDIF file named rootdn.ldif Replace xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx with the hashed password of our LDAP administrator:

```bash
sudo nano rootdn.ldif
```

5. Use the ldapadd utility to add the new root DN and password to the OpenLDAP database:

```bash
ldapadd -x -D "cn=admin,dc=example,dc=example" -w <our-password> -f rootdn.ldif
```

```text
dn: cn=admin,dc=logsrlife,dc=example
objectClass: simpleSecurityObject
objectClass: organizationalRole
cn: admin
userPassword: {SSHA}xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

6. Edit the `slapd.conf` file 
```bash
sudo nano /etc/openldap/slapd.conf
```

**Note**: For an actual environment we would likely want to make the following amendments to the config:

   1. suffix: Replace dc=logsrlife,dc=example with our own domain name exampleponents separated by ,dc=.
   2. `rootdn`: Replace admin,dc=logsrlife,dc=example with the DN of our LDAP administrator account.
   3. `rootpw`: Replace xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx with the hashed password of our LDAP administrator account .
   4. `directory`: Replace /var/lib/ldap with the path to the directory where we want to store our LDAP database.
   5. `TLSCACertificateFile`: Replace /etc/openldap/certs/ca-cert.pem with the path to our CA certificate file.
   6. `TLSCertificateFile`: Replace /etc/openldap/certs/server-cert.pem with the path to our server certificate file.
   7. `TLSCertificateKeyFile`: Replace /etc/openldap/certs/server-key.pem with the path to our server private key file.

**Note**: Indexes in OpenLDAP are used to speed up searches by pre-sorting the data. The index directive in slapd.conf is used to specify which attributes should be indexed.

**Note**: There are four attributes that are indexed: objectClass, cn, sn, and uid. The eq keyword means that exact matching is used for the attribute, while pres and sub refer to present and substring matching respectively.

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
TLSCACertificateFile /etc/ssl/certs/logsrlife-ca.crt
TLSCertificateFile /etc/ssl/certs/logsrlife.crt
TLSCertificateKeyFile /etc/ssl/private/logsrlife.key
```

```bash
# Start and enable the service
sudo systemctl start slapd
sudo systemctl enable slapd
```


```bash
# Add Group to LDAP
sudo ldapadd -x -D "cn=admin,dc=logsrlife,dc=example" -w admin_password <<EOF
dn: cn=security_operations_analyst,ou=Groups,dc=logsrlife,dc=example
objectClass: groupOfNames
cn: security_operations_analyst
member: uid=logscale_analyst,ou=staff,dc=logsrlife,dc=example
EOF
```
We'll use the same hashed password for our analyst user for ease of lab

```bash
# Add User to New Group in LDAP
sudo ldapadd -x -D "cn=admin,dc=logsrlife,dc=example" -w admin_password <<EOF
dn: uid=logscale_analyst,ou=staff,dc=logsrlife,dc=example
objectClass: inetOrgPerson
objectClass: posixAccount
uid: logscale_analyst
cn: Log Scale Analyst
sn: Analyst
userPassword: {SSHA}xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
uidNumber: 10001
gidNumber: 10001
homeDirectory: /home/logscale_analyst
loginShell: /bin/bash
EOF
```
