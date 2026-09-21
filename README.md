# HashiCorp Vault Lab

This lab covers the basic installation and configuration of HashiCorp Vault.

## Tasks

* Task 1 - Install and Initialize Vault
* Task 2 - Unseal Vault and Configure Authentication
* Task 3 - Enable KV Secrets Engine
* Task 4 - Store and Retrieve Secrets
* Task 5 - Create Policy and Verify Access

---

## Task 1 - Install and Initialize Vault

Check the Vault version:

```bash
vault --version
```

Check the Vault configuration:

```bash
sudo cat /etc/vault.d/vault.hcl
```

Start the Vault service:

```bash
sudo systemctl start vault
```

Check the service:

```bash
sudo systemctl status vault
```

Check the Vault status:

```bash
vault status
```

Initialize Vault:

```bash
vault operator init
```

After initialization, Vault provides unseal keys and a root token.

The unseal keys and root token were saved separately and were not added to the Git repository.

---

## Task 2 - Unseal Vault and Configure Authentication

Check the Vault status:

```bash
vault status
```

Unseal Vault:

```bash
vault operator unseal
```

Enter the required unseal keys when prompted.

Check the status again:

```bash
vault status
```

Vault should now show:

```text
Sealed: false
```

Login using the root token:

```bash
vault login
```

Enable Userpass authentication:

```bash
vault auth enable userpass
```

Create a Userpass user:

```bash
vault write auth/userpass/users/vault \
    password="YOUR_PASSWORD" \
    token_policies="creds-read"
```

Login with the Userpass user:

```bash
vault login -method=userpass
```

---

## Task 3 - Enable KV Secrets Engine

Enable KV version 2:

```bash
vault secrets enable -path=secret kv-v2
```

Check the enabled secrets engines:

```bash
vault secrets list -detailed
```

The `secret/` mount should show KV version 2.

---

## Task 4 - Store and Retrieve Secrets

Store a secret:

```bash
vault kv put -mount=secret creds passcode="YOUR_SECRET"
```

Check the secret:

```bash
vault kv get -mount=secret creds
```

To get only the passcode:

```bash
vault kv get -mount=secret -field=passcode creds
```

Check the metadata:

```bash
vault kv metadata get -mount=secret creds
```

---

## Task 5 - Create Policy and Verify Access

Create the policy file:

```bash
nano creds-read.hcl
```

Add the following:

```hcl
path "secret/data/creds" {
  capabilities = ["read"]
}
```

Save the file and create the policy:

```bash
vault policy write creds-read creds-read.hcl
```

Check the policy:

```bash
vault policy read creds-read
```

Attach the policy to the Userpass user:

```bash
vault write auth/userpass/users/vault \
    token_policies="creds-read"
```

Check the user configuration:

```bash
vault read auth/userpass/users/vault
```

Logout:

```bash
vault logout
```

Login again using the Userpass account:

```bash
vault login -method=userpass
```

Check the token:

```bash
vault token lookup
```

The `creds-read` policy should be listed.

### Test Read Access

```bash
vault kv get -mount=secret creds
```

The secret should be displayed.

### Test Write Access

Try to change the secret:

```bash
vault kv put -mount=secret creds passcode="test-value"
```

The operation should be denied because the policy only has read permission.

### Check Capabilities

```bash
vault token capabilities secret/data/creds
```

Expected output:

```text
read
```

This confirms that the user has read access to the secret.

---

## Final Checks

Check Vault status:

```bash
vault status
```

Check authentication methods:

```bash
vault auth list
```

Check secrets engines:

```bash
vault secrets list
```

Check policies:

```bash
vault policy list
```

Check the current token:

```bash
vault token lookup
```

---
# Harbor-Setup
# Harbor Setup and Configuration

## 1. Install Harbor

Update packages and install Docker:

We use Ansible Playbook to install docker and docker compose on this task. 
docker-install.yaml -->
---
- name: Install Docker and Docker Compose
  hosts: task-servers
  become: true
  tasks:
    - name: Update the Cache
      ansible.builtin.apt:
        update_cache: yes

    - name: Install Required Packages
      ansible.builtin.apt:
        name:
          - curl
          - ca-certificates
          - git
          - bc
        state: present

    - name: Create docker keyrings directory
      ansible.builtin.file:
        path: /etc/apt/keyrings
        state: directory
        mode: '0755'

    - name: Add Docker GPG key
      ansible.builtin.get_url:
        url: https://download.docker.com/linux/ubuntu/gpg
        dest: /etc/apt/keyrings/docker.asc
        mode: '0644'

    - name: Add Docker Repository
      ansible.builtin.apt_repository:
        repo: "deb [arch=amd64 signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu {{ ansible_distribution_release }} stable"
        state: present
        filename: docker
        update_cache: yes

    - name: Install Docker Engine
      ansible.builtin.package:
        name:
          - docker-ce
          - docker-ce-cli
          - containerd.io
          - docker-buildx-plugin
          - docker-compose-plugin
        state: present
      notify:
        - Ensure Docker Service is Enabled and Started

    - name: Add Ubuntu user into docker group
      ansible.builtin.user:
        name: ubuntu
        groups: docker
        append: yes

  handlers:
    - name: Ensure Docker Service is Enabled and Started
      ansible.builtin.service:
        name: docker
        enabled: true
        state: started

Download and extract Harbor:

```bash
cd /home/ubuntu
tar -xvf harbor-offline-installer-v2.15.2.tgz
cd harbor
```

Copy the configuration file:

```bash
sudo cp harbor.yml.tmpl harbor.yml
sudo nano harbor.yml
```

Configure:

```yaml
hostname: harbor.local

https:
  port: 443
  certificate: /data/cert/harbor.local.crt
  private_key: /data/cert/harbor.local.key

harbor_admin_password: <password>

data_volume: /data
```

---

## 2. Configure HTTPS

Create the certificate directory:

```bash
sudo mkdir -p /data/cert
cd /data/cert
```

Create a CA:

```bash
sudo openssl genrsa -out ca.key 4096

sudo openssl req -x509 -new -nodes \
  -key ca.key \
  -sha256 \
  -days 3650 \
  -out ca.crt \
  -subj "/C=IN/O=DevOps/CN=Harbor-CA"
```

Create Harbor certificate:

```bash
sudo openssl genrsa -out harbor.local.key 4096

sudo openssl req -new \
  -key harbor.local.key \
  -out harbor.local.csr \
  -subj "/C=IN/O=DevOps/CN=harbor.local"
```

Create SAN configuration:

```bash
sudo nano /data/cert/v3.ext
```

Add:

```text
authorityKeyIdentifier=keyid,issuer
basicConstraints=CA:FALSE
keyUsage=digitalSignature,keyEncipherment
extendedKeyUsage=serverAuth
subjectAltName=@alt_names

[alt_names]
DNS.1=harbor.local
DNS.2=harbor
IP.1=<HARBOR_PRIVATE_IP>
```

Generate certificate:

```bash
sudo openssl x509 -req \
  -in harbor.local.csr \
  -CA ca.crt \
  -CAkey ca.key \
  -CAcreateserial \
  -out harbor.local.crt \
  -days 825 \
  -sha256 \
  -extfile v3.ext
```

Verify:

```bash
sudo openssl verify \
  -CAfile ca.crt harbor.local.crt
```

---

## 3. Configure Hostname

Add Harbor IP to `/etc/hosts`:

```bash
sudo nano /etc/hosts
```

```text
<HARBOR_PRIVATE_IP> harbor.local harbor
```

Verify:

```bash
getent hosts harbor.local
```

---

## 4. Start Harbor

Generate Harbor configuration:

```bash
cd /home/ubuntu/harbor
sudo ./prepare
```

Start Harbor:

```bash
sudo docker compose up -d
```

Check containers:

```bash
sudo docker compose ps
```

All Harbor containers should show `healthy`.

Test HTTPS:

```bash
curl -k -I https://harbor.local
```

---

## 5. Create Project and User

Open:

```text
https://harbor.local
```

Login as `admin`.

Create project:

```text
Projects → New Project
Project Name: myproject
Access: Private
```

Create a user:

```text
Administration → Users → New User
```

Add the user to `myproject` with the **Developer** role.

---

## 6. Push Docker Image

Login:

```bash
docker login harbor.local
```

Build image:

```bash
docker build -t harbor-test:1 .
```

Tag image:

```bash
docker tag harbor-test:1 harbor.local/myproject/harbor-test:1
```

Push:

```bash
docker push harbor.local/myproject/harbor-test:1
```

---

## 7. Pull Docker Image

Remove the local image:

```bash
docker rmi harbor.local/myproject/harbor-test:1
```

Pull from Harbor:

```bash
docker pull harbor.local/myproject/harbor-test:1
```

Verify:

```bash
docker images | grep harbor-test
```

---

## 8. Verify in Harbor

Open:

```text
Projects → myproject → Repositories
```

Verify:

```text
Repository: harbor-test
Tag: 1
```

The final image name is:

```text
harbor.local/myproject/harbor-test:1
```

Harbor is now configured with HTTPS, a project and user, and Docker image push/pull has been tested.

