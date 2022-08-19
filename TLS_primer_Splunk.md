# Create a local PKI server and distribute certificates to a demo Splunk server

Authentication of the server (server is who it says it is), and optional authentication of the client are requirements of zero trust architecture. We also want to achieve bulk encryption of data in transit. There are several moving parts, "CA", "keys", "CSRs", and the "certs". We often say "SSL" when we mean "TLS". SSL is effectively deprecated.

Out of the box:
        - All certificates are generated on a default-shipped CA configuration
        - Splunkweb does not use SSL
        - Splunkd uses SSL for the REST port - with certificate verification **disabled**
        - No SSL data inputs/outputs are defined
        - Splunkd LDAP can use SSL - with certificate verification **disabled**


### Demo CA Creation
```bash
#Generate the CA private key file
openssl genrsa -out demo-ca.key 2048
#Provision the CA
openssl req -x509 -new -nodes\
  -key       demo-ca.key -sha256 -subj "/C=US/ST=MI/L=Detroit/O=Detroit Cyber/OU=Cybersecurity/CN=demo-ca"\
        -days 365 -out demo-ca.crt #1 year CA

```
### Splunk Cert Signing Request (CSR) Configuration Creation
Create a configuration file named <server>csr.conf for generating the Certificate Signing Request (CSR) as shown below. Replace the values as appropriate. 



```bash
cat > demo-splunk1.csr.conf <<EOF
[ req ]
default_bits = 2048
prompt = no
default_md = sha256
req_extensions = req_ext
distinguished_name = dn

[ dn ]
C = US
ST = Michigan
L = Detroit
O = Detroit Cyber
OU = Cybersecurity
CN = demo-splunk1.turnerhomestead.com

[ req_ext ]
subjectAltName = @alt_names

[v3_req]
subjectAltName = @alt_names
extendedKeyUsage = serverAuth
keyUsage = keyEncipherment, dataEncipherment

[ alt_names ]
DNS.1   = demo-splunk1.turnerhomestead.com
DNS.2   = demo-splunk1
DNS.3   = *.demo-splunk1.turnerhomestead.com
IP.1    = 192.168.86.193


EOF
```
### Splunk server private key creation

```bash
openssl genrsa -out demo-splunk1.key 2048
```

### Splunk server cert signing request (CSR) creation

```bash
openssl req -new -key demo-splunk1.key -out demo-splunk1.csr -config demo-splunk1.csr.conf
```
### Generate the Splunk server SSL certificate using ca.key, ca.crt and server.csr
```
openssl x509 -req -in demo-splunk1.csr -CA demo-ca.crt -CAkey demo-ca.key  -extfile demo-splunk1.csr.conf -CAcreateserial -out demo-splunk1.crt -days 90 # 3-month 
```

### Operations

To upload custom SSL certs to install with Splunk for the Web UI (default tcp/8000):
Upload the certs to $SPLUNK_HOME/etc/auth/my-certs/ on the splunk hosts and perform default configuration to reference these certs in $SPLUNK_HOME/etc/system/local/web.conf

```bash
# Concatenate the demo-splunk1.crt with the Root CA crt, so that Splunk trusts the Root CA and it's issued certificates
# Copy over the certificate the the appropriate directory and file
cat demo-splunk1.crt demo-ca.crt > demo-splunk1.bun.crt && cp demo=splunk1.bun.crt $SPLUNK_HOME/etc/auth/mycerts/cert.pem 
# 
cp demo-splunk1.key $SPLUNK_HOME/etc/auth/mycerts/privkey.pem  
```

Once you are savy enough try implementing an automation via ansible. Your Role tasks should look something like the following:

```yaml
---

- name: Create {{ splunk_home }}/etc/auth/mycerts dir
  file:
    path: "{{ splunk_home }}/etc/auth/mycerts/"
    state: directory
    mode: '0755'  
    owner: "{{ os_user }}"
    group: "{{ os_group }}"
  become: yes

- name: -SSL Certs- Upload public key file (mycerts/cert.pem)
  copy:
    src: "certs/cert.pem"
    dest: "{{ splunk_home }}/etc/auth/mycerts/cert.pem"
    mode: '0640'
    owner: "{{ os_user }}"
    group: "{{ os_group }}"
  become: yes

- name: -SSL Certs- Upload private key file (mycerts/privkey.pem)
  copy:
    src: "certs/privkey.pem"
    dest: "{{ splunk_home }}/etc/auth/mycerts/privkey.pem"
    mode: '0640'
    owner: "{{ os_user }}"
    group: "{{ os_group }}"
  become: yes

- name: -config updates- Ensure $SPLUNK_HOME/etc/system/local/web.conf exists
  file:
    path: "{{ splunk_home }}/etc/system/local/web.conf"
    state: file
    owner: "{{ os_user }}"
    group: "{{ os_group }}"
  become: yes

- name: -config updates- Ensure web.conf contains the [settings] stanza
  lineinfile:
    path: "{{ splunk_home }}/etc/system/local/web.conf"
    line: "[settings]"
    state: present
  become: yes

- name: -server.conf- Ensure file contains [sslConfig] stanza
  template:
    src: server.conf.j2
    dest: "{{ splunk_home }}/etc/system/local/server.conf"
  become: yes
  tags:
    - ssl-enable

- name: -config updates- Add serverCert attribute to [settings]
  lineinfile:
    path: "{{ splunk_home }}/etc/system/local/web.conf"
    regexp: '^serverCert = {{ splunk_home }}/etc/auth/mycerts/cert.pem'
    insertbefore: '^[settings]'
    line: 'serverCert = {{ splunk_home }}/etc/auth/mycerts/cert.pem'    
  become: yes

- name: -config updates- Add privKeyPath attribute to [settings]
  lineinfile:
    path: "{{ splunk_home }}/etc/system/local/web.conf"
    regexp: '^privKeyPath = {{ splunk_home }}/etc/auth/mycerts/privkey.pem'
    insertbefore: '^[settings]'
    line: 'privKeyPath = {{ splunk_home }}/etc/auth/mycerts/privkey.pem'    
  become: yes

- name: -config updates- Set enableSplunkWebSSL = 1
  replace:
    path: "{{ splunk_home }}/etc/system/local/web.conf"
    after: '[settings]'
    regexp: 'enableSplunkWebSSL = 0'
    replace: 'enableSplunkWebSSL = 1'
  become: yes
  notify: restart_splunk
        
```

Your server.conf.j2 should look something like this:
```jinja
[sslConfig]
sslVerifyServerCert = true # turns on TLS certificate requirements
sslVerifyServerName = true # turns on TLS certificate host name validation
cliVerifyServerName = true # turns on TLS certificate host name validation
sslRootCaPath = /opt/splunk/etc/auth/mycerts/cert.pem        
```
