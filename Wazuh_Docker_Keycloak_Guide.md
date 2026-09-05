## Wazuh Docker Setup with Keycloak SSO and UFW Integration

This guide documents the complete setup of Wazuh 4.14.7 on Docker with Keycloak Single Sign-On (SSO) integration and UFW firewall configuration. I had sevral issues with configuring Keycloak and UFW since I'm using Doker, and I'd want to show you how I fixed them.

[Threat Detection and Monitoring Lab](https://github.com/AljawharaK/SIEM-Threat-Detection-and-Automated-Response-Lab) | [Official_Keycloak_Guide](https://documentation.wazuh.com/current/user-manual/user-administration/single-sign-on/keycloak.html#setup-keycloak-single-sign-on-with-administrator-role)

---

## Prerequisites

*   Ubuntu server with Docker installed
    
*   Administrative access to the server
    
*   Required ports: 443 (Dashboard), 1514/TCP (events), 1515/TCP (enrollment), 8081 (Keycloak)
    
---

## System Preparation

Update your system and install required packages:

```bash
sudo apt update
sudo apt install -y git curl
```  

### Configure Kernel Parameters

The Wazuh Indexer (OpenSearch) requires specific kernel settings:

```bash
sudo sysctl -w vm.max_map_count=262144
```

### Install Docker

Add Docker's official repository and install Docker:

```bash
sudo apt install ca-certificates curl -y
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt update
sudo apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin -y

sudo systemctl enable docker
sudo systemctl start docker
```

Verify Docker installation:

```bash
docker --version
docker compose version
``` 

---

## Download and Prepare Wazuh

Clone the Wazuh Docker repository (version 4.14.7):

```bash
git clone https://github.com/wazuh/wazuh-docker.git -b v4.14.7
cd wazuh-docker/single-node
``` 

### Generate SSL Certificates

Generate self-signed certificates for the Wazuh stack:

```bash
sudo docker compose -f generate-indexer-certs.yml run --rm generator
sudo chown -R 1000:1000 config/wazuh_indexer_ssl_certs/
``` 

### Initial Stack Startup

Start the Wazuh stack:

```bash
sudo docker compose up -d
```

Run this command on your Ubuntu host to find its IP address:

```bash
hostname -I | awk '{print $1}'
```

You can access Wazuh on https://Your_IP

**Default credentials:**

*   Dashboard login: admin / SecretPassword
    
*   API user: wazuh-wui / MyS3cr37P450r.\*-
    

> **Note:** Browser warnings are expected due to self-signed certificates.

---

## Change Default Passwords

### 1\. Stop the Stack

```bash
sudo docker compose down
```

### 2\. Generate New Password Hashes

Generate hashes for your new passwords:

```bash
sudo docker run --rm -ti wazuh/wazuh-indexer:4.14.7 \
  bash /usr/share/wazuh-indexer/plugins/opensearch-security/tools/hash.sh
```

### 3\. Update Internal Users Configuration

Edit config/wazuh\_indexer/internal\_users.yml:

```bash
nano config/wazuh_indexer/internal_users.yml
```

Replace the hash values for admin and kibanaserver:

```bash
---
_meta:
  type: "internalusers"
  config_version: 2

admin:
  hash: "<Your_Pass_Hash>"
  reserved: true
  backend_roles:
  - "admin"
  description: "Demo admin user"

kibanaserver:
  hash: "<Your_Pass_Hash>"
  reserved: true
  description: "Demo kibanaserver user"
``` 

### 4\. Update Environment Variables

Edit docker-compose.yml to update passwords:

```bash
nano docker-compose.yml
```

Update the environment sections for wazuh.manager and wazuh.dashboard:

```bash
wazuh.manager:
    environment:
      - INDEXER_USERNAME=admin
      - INDEXER_PASSWORD=<Your_Password>
      - API_PASSWORD=<Your_Password>.*-
      
wazuh.dashboard:
    environment:
      - INDEXER_USERNAME=admin
      - INDEXER_PASSWORD=<Your_Password>
      - API_PASSWORD=<Your_Password>.*-
``` 

### 5\. Update Wazuh Dashboard Configuration

Edit config/wazuh\_dashboard/wazuh.yml:

```bash
nano config/wazuh_dashboard/wazuh.yml
``` 

Update with your new API password:

```bash
hosts:
  - 1513629884013:
      url: https://wazuh.manager
      port: 55000
      username: wazuh-wui
      password: <Your_Password>.*-
      run_as: true
```

### 6\. Apply Changes

Restart the stack and apply security configuration:

```bash
sudo docker compose up -d

sudo docker exec -it single-node-wazuh.indexer-1 bash -lc '
  export JAVA_HOME=/usr/share/wazuh-indexer/jdk;
  bash /usr/share/wazuh-indexer/plugins/opensearch-security/tools/securityadmin.sh \
    -cd /usr/share/wazuh-indexer/config/opensearch-security/ -nhnv \
    -cacert /usr/share/wazuh-indexer/config/certs/root-ca.pem \
    -cert /usr/share/wazuh-indexer/config/certs/admin.pem \
    -key /usr/share/wazuh-indexer/config/certs/admin-key.pem \
    -p 9200 -icl'
```

---

## Configure UFW Firewall

### 1\. Install UFW Docker Integration

Install the UFW Docker helper script:

```bash
sudo wget -O /usr/local/bin/ufw-docker \
  https://github.com/chaifeng/ufw-docker/raw/master/ufw-docker
sudo chmod +x /usr/local/bin/ufw-docker
```

### 2\. Configure iptables Legacy Mode

```bash
sudo update-alternatives --set iptables /usr/sbin/iptables-legacy
sudo update-alternatives --set ip6tables /usr/sbin/ip6tables-legacy
iptables --version
```

### 3\. Add Docker Firewall Rules

Edit /etc/ufw/after.rules:

```bash
sudo nano /etc/ufw/after.rules
```

Add the following configuration at the end of the file (after the existing COMMIT line):

```bash
# BEGIN UFW AND DOCKER
*filter
:ufw-user-forward - [0:0]
:DOCKER-USER - [0:0]
-A DOCKER-USER -j RETURN -s 10.0.0.0/8
-A DOCKER-USER -j RETURN -s 172.16.0.0/12
-A DOCKER-USER -j RETURN -s 192.168.0.0/16

-A DOCKER-USER -j ufw-user-forward

-A DOCKER-USER -j DROP -p tcp -m tcp --tcp-flags FIN,SYN,RST,ACK SYN -d 192.168.0.0/16
-A DOCKER-USER -j DROP -p tcp -m tcp --tcp-flags FIN,SYN,RST,ACK SYN -d 10.0.0.0/8
-A DOCKER-USER -j DROP -p tcp -m tcp --tcp-flags FIN,SYN,RST,ACK SYN -d 172.16.0.0/12
-A DOCKER-USER -j DROP -p udp -m udp --dport 0:32767 -d 192.168.0.0/16
-A DOCKER-USER -j DROP -p udp -m udp --dport 0:32767 -d 10.0.0.0/8
-A DOCKER-USER -j DROP -p udp -m udp --dport 0:32767 -d 172.16.0.0/12

-A DOCKER-USER -j RETURN
COMMIT
# END UFW AND DOCKER
```

### 4\. Apply Firewall Rules

```bash
sudo ufw-docker install
sudo systemctl restart ufw
```

### 5\. Allow Required Ports

Allow access to Wazuh containers:

```bash
sudo ufw-docker allow single-node-wazuh.manager-1 1514/tcp
sudo ufw-docker allow single-node-wazuh.manager-1 1515/tcp
sudo ufw-docker allow single-node-wazuh.manager-1 55000/tcp
sudo ufw-docker allow single-node-wazuh.dashboard-1 5601/tcp
sudo ufw-docker allow single-node-wazuh.dashboard-1 443/tcp
```

---

## Integrate Keycloak SSO

### 1\. Add Keycloak to Docker Compose

Edit the docker-compose.yml file:

```bash
nano docker-compose.yml
```

Add Keycloak service configuration:

```bash
services:
  wazuh.manager:
    # ... existing configuration ...
    environment:
      - INDEXER_URL="https://wazuh.indexer:9200"
      - INDEXER_USERNAME=admin
      - INDEXER_PASSWORD=<Your_Password>
      - FILEBEAT_SSL_VERIFICATION_MODE=full
      - SSL_CERTIFICATE_AUTHORITIES=/etc/ssl/root-ca.pem
      - SSL_CERTIFICATE=/etc/ssl/filebeat.pem
      - SSL_KEY=/etc/ssl/filebeat.key
      - API_USERNAME=wazuh-wui
      - API_PASSWORD=<Your_Password>.*-
    # ... volumes configuration ...

  wazuh.indexer:
    # ... existing configuration ...

  wazuh.dashboard:
    # ... existing configuration ...
    environment:
      - INDEXER_USERNAME=admin
      - INDEXER_PASSWORD=<Your_Password>
      - WAZUH_API_URL=https://wazuh.manager
      - DASHBOARD_USERNAME=kibanaserver
      - DASHBOARD_PASSWORD=kibanaserver
      - API_USERNAME=wazuh-wui
      - API_PASSWORD=<Your_Password>.*-
    # ... volumes configuration ...

  keycloak:
    image: keycloak/keycloak:26.0.5-0
    container_name: keycloak
    restart: unless-stopped
    environment:
      - KC_DB=dev-file
      - KC_HTTP_ENABLED=true
      - KEYCLOAK_ADMIN=admin
      - KEYCLOAK_ADMIN_PASSWORD=<Your_Password>
    volumes:
      - keycloak_data:/opt/keycloak/data
    ports:
      - 8081:8080
    command:
      - start-dev

volumes:
  # ... existing volumes ...
  keycloak_data:
```

Start Keycloak:

```bash
sudo docker compose up -d keycloak
```

### 2\. Configure Keycloak

Access Keycloak admin console at http://<Your_IP>:8081

#### Create a New Realm

1.  Click on **Manage realms** > **Create realm**
    
2.  Realm name: Wazuh
    
3.  Click **Create**
    

#### Create a New Client

1.  Navigate to **Clients** > **Create client**
    
2.  Client type: **SAML**
    
3.  Client ID: wazuh-saml (This is the SP Entity ID)
    
4.  Click **Next** and **Save**
    

#### Configure Client Settings

**a. General Settings:**

*   Navigate to **Clients** > **Settings**
    
*   Ensure **Enabled** is ON
    
*   Client ID: wazuh-saml
    
*   Name: Wazuh SSO
    
*   Valid redirect URIs: https://<Your_IP>/*
    
*   IDP-Initiated SSO URL name: wazuh-dashboard
    
*   Name ID format: username
    
*   Force POST binding: **ON**
    
*   Include AuthnStatement: **ON**
    
*   Sign documents: **ON**
    
*   Sign assertions: **ON**
    
*   Signature algorithm: **RSA\_SHA256**
    
*   SAML signature key name: **KEY\_ID**
    
*   Canonicalization method: **EXCLUSIVE**
    
*   Front channel logout: **ON**
    
*   Click **Save**
    

**b. Keys Settings:**

*   Navigate to **Clients** > **Keys**
    
*   Client signature required: **Off**
    

**c. Advanced Settings:**

*   Navigate to **Clients** > **Advanced** > **Fine Grain SAML Endpoint Configuration**
    
*   Assertion Consumer Service POST Binding URL: https://<Your_IP>/_opendistro/_security/saml/acs/idpinitiated
    
*   Logout Service Redirect Binding URL: https://<Your_IP>
    
*   Click **Save**
    

#### Create a New Role

1.  Navigate to **Realm roles** > **Create role**
    
2.  Role name: wazuh-admins
    
3.  Click **Save**
    

#### Create a New User

1.  Navigate to **Users** > **Add user**
    
2.  Fill in the required information
    
3.  Click **Create**
    
4.  Go to **Users** > **Credentials** > **Set password**
    
5.  Set a password and click **Save**
    

#### Create a New Group

1.  Go to **Groups** > **Create group**
    
2.  Group name: Wazuh-admins
    
3.  Click **Create**
    
4.  Click on the group, navigate to **Members** > **Add member**
    
5.  Select the user created earlier and click **Add**
    
6.  Go to **Role Mapping** > **Assign role** > **Realm roles**
    
7.  Select wazuh-admins and click **Assign**
    

#### Configure Protocol Mapper

1.  Navigate to **Client scopes** > **role\_list** > **Mappers**
    
2.  Click **Add mapper** > **By configuration**
    
3.  Select **Role list**
    
4.  Fill in:
    
    *   Mapper type: **Role list**
        
    *   Name: wazuhRoleKey
        
    *   Role attribute name: Roles
        
    *   SAML Attribute NameFormat: **Basic**
        
    *   Single Role Attribute: **On**
        
5.  Click **Save**
    

**Important:** Remove the default role list mapper to solve "Found an Attribute element with duplicated Name" error.

#### Download SAML Metadata

1.  Navigate to **Clients** and select wazuh-saml
    
2.  Go to **Action** > **Download adaptor config**
    
3.  Format: **mod-auth-mellon**
    
4.  Click **Download** to get idp-metadata.xml and sp-metadata.xml
    

### 3\. Configure Wazuh Indexer for SAML

#### Prepare SAML Files

```bash
mkdir -p ~/wazuh-docker/saml-config
# Copy downloaded idp-metadata.xml and sp-metadata.xml to ~/wazuh-docker/saml-config/
```

#### Generate Exchange Key

```bash
openssl rand -hex 32
# Example output: da01ba6af157ef83fa0338d77ca42d6a5025cd77d848d36d18875f825933bcc7
```

#### Copy Files to Indexer Container

```bash
sudo docker cp ~/wazuh-docker/saml-config/idp-metadata.xml single-node-wazuh.indexer-1:/usr/share/wazuh-indexer/config/opensearch-security/
sudo docker cp ~/wazuh-docker/saml-config/sp-metadata.xml single-node-wazuh.indexer-1:/usr/share/wazuh-indexer/config/opensearch-security/
sudo docker exec -it single-node-wazuh.indexer-1 chown wazuh-indexer:wazuh-indexer /usr/share/wazuh-indexer/config/opensearch-security/idp-metadata.xml
sudo docker exec -it single-node-wazuh.indexer-1 chown wazuh-indexer:wazuh-indexer /usr/share/wazuh-indexer/config/opensearch-security/sp-metadata.xml
```

#### Update config.yml

```bash
sudo docker cp single-node-wazuh.indexer-1:/usr/share/wazuh-indexer/config/opensearch-security/config.yml ./config.yml
sudo nano ./config.yml
```

Update the authentication section:

```bash
authc:
  basic_internal_auth_domain:
    description: "Authenticate via HTTP Basic against internal users database"
    http_enabled: true
    transport_enabled: true
    order: 0   # MUST BE 0
    http_authenticator:
      type: "basic"
      challenge: false
    authentication_backend:
      type: "intern"
  saml_auth_domain:
    http_enabled: true
    transport_enabled: false
    order: 1   # MUST BE 1 or higher than basic domain
    http_authenticator:
      type: saml
      challenge: true
      config:
        idp:
          metadata_file: '/usr/share/wazuh-indexer/config/opensearch-security/idp-metadata.xml'
          entity_id: 'http://<Your_IP>:8081/realms/Wazuh'
        sp:
          entity_id: wazuh-saml
          metadata_file: '/usr/share/wazuh-indexer/config/opensearch-security/sp-metadata.xml'
        kibana_url: https://<Your_IP>
        roles_key: Roles
        exchange_key: 'da01ba6af157ef83fa0338d77ca42d6a5025cd77d848d36d18875f825933bcc7' # The hash you created
    authentication_backend:
      type: noop
```

Apply the configuration:

```bash
sudo docker cp ./config.yml single-node-wazuh.indexer-1:/usr/share/wazuh-indexer/config/opensearch-security/config.yml
sudo docker exec -it --user root single-node-wazuh.indexer-1 chown wazuh-indexer:wazuh-indexer /usr/share/wazuh-indexer/config/opensearch-security/config.yml
```

#### Update roles\_mapping.yml

```bash
sudo docker cp single-node-wazuh.indexer-1:/usr/share/wazuh-indexer/config/opensearch-security/roles_mapping.yml ./roles_mapping.yml
sudo nano ./roles_mapping.yml
```

Replace with:

```bash
all_access:
  reserved: false
  hidden: false
  backend_roles:
  - "wazuh-admins"
  users:
  - "admin"  # Must add the admin user to avoid errors
  description: "Maps admin to all_access"
```

Apply the configuration:

```bash
sudo docker cp ./roles_mapping.yml single-node-wazuh.indexer-1:/usr/share/wazuh-indexer/config/opensearch-security/roles_mapping.yml
sudo docker exec -it --user root single-node-wazuh.indexer-1 chown wazuh-indexer:wazuh-indexer /usr/share/wazuh-indexer/config/opensearch-security/roles_mapping.yml
```

#### Apply Security Changes

Run the security admin tool three times to apply all changes:

```bash
sudo docker exec -it --user root single-node-wazuh.indexer-1 bash -c 'export JAVA_HOME=/usr/share/wazuh-indexer/jdk/ && bash /usr/share/wazuh-indexer/plugins/opensearch-security/tools/securityadmin.sh -f /usr/share/wazuh-indexer/config/opensearch-security/config.yml -icl -key /usr/share/wazuh-indexer/config/certs/admin-key.pem -cert /usr/share/wazuh-indexer/config/certs/admin.pem -cacert /usr/share/wazuh-indexer/config/certs/root-ca.pem -h localhost -nhnv'

sudo docker exec -it --user root single-node-wazuh.indexer-1 bash -c 'export JAVA_HOME=/usr/share/wazuh-indexer/jdk/ && bash /usr/share/wazuh-indexer/plugins/opensearch-security/tools/securityadmin.sh -f /usr/share/wazuh-indexer/config/opensearch-security/roles_mapping.yml -icl -key /usr/share/wazuh-indexer/config/certs/admin-key.pem -cert /usr/share/wazuh-indexer/config/certs/admin.pem -cacert /usr/share/wazuh-indexer/config/certs/root-ca.pem -h localhost -nhnv'
```

### 4\. Configure Wazuh Dashboard for SAML

#### Create Role Mapping in Dashboard

1.  Click ☰ to open the menu on the Wazuh dashboard
    
2.  Go to **Server management** > **Security** > **Roles mapping**
    
3.  Click **Create Role mapping** and fill in:
    
    *   Role mapping name: Assign a name
        
    *   Roles: Select **administrator**
        
    *   Custom rules:
        
        *   User field: backend\_roles
            
        *   Search operation: **FIND**
            
        *   Value: wazuh-admins
            

#### Update Dashboard Configuration

```bash
cd ~/wazuh-docker/single-node/config/wazuh_dashboard/
sudo nano opensearch_dashboards.yml
```

Add these lines:

```bash
opensearch_security.auth.multiple_auth_enabled: true
opensearch_security.auth.type: ["basicauth","saml"]
server.xsrf.allowlist: ["/_opendistro/_security/saml/acs", "/_opendistro/_security/saml/logout", "/_opendistro/_security/saml/acs/idpinitiated"]
```

#### Verify Wazuh API Configuration

```bash
sudo nano wazuh.yml
```

Ensure the file contains:

```bash
hosts:
  - 1513629884013:
      url: "https://wazuh.manager"
      port: 55000
      username: wazuh-wui
      password: "<Your_Password>.*-"
      run_as: true # Must be set to true
```

#### Restart Dashboard

```bash
sudo docker start single-node-wazuh.dashboard-1
sudo docker restart single-node-wazuh.dashboard-1
```

---
## Enroll a Linux Agent


In the dashboard, open the menu, then **Agents Management** -> **Summary**, then **Deploy new agent**. Choose Linux, copy the commands and run them on the client.

Verify the agent shows **Active** in the dashboard or using command:

```bash
sudo docker exec single-node-wazuh.manager-1 /var/ossec/bin/agent_control -l
```

### Quick Test: Generate Events

On the enrolled Linux host:

```bash
# File system access
sudo cat /etc/shadow
# Nmap scan
nmap -sV -p 443, 1514, 1515 <Your_IP>
```

In the dashboard, open **Threat Hunting** -> **Events**, filter by your agent and confirm logs arrive.

---

## Next Steps

*   Configure notifications (email and Slack)
  
*   Set up additional security monitoring for other services (websites, databases, etc.)
    
*   Tune rules and decoders in Wazuh Dashboard
    
*   Configure index lifecycle management

---
