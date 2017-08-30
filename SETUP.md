# POC Vault by [HashiCorp](https://www.vaultproject.io) - Regulated SSH and MySQL Access
## Consul setup
### Download Consul
Download consul's ZIP file to install the one binary that's provided in the binary release:
```bash
wget https://releases.hashicorp.com/consul/0.9.3/consul_0.9.3_linux_amd64.zip
```
or go to [HashiCorp's Consul](https://www.consul.io/downloads.html) download page for the latest version if so desired.

#### Installation of the binary
Next step is to unzip the downloaded file and installing it into the existing PATH of the system
(adjust for a different version that you might have downloaded)
```
unzip consul_0.9.3_linux_amd64.zip
```

Ensure that the consul file is executable
```bash
chmod +x consul
```

Move the binary `consul` into `/usr/bin`
```bash
mv consul /usr/bin/
```
The consul service doesn't need a configuration file per se, but it needs a data directory
```bash
mkdir /opt/consuldata
```

## Vault setup
### Download Vault
Download vault's ZIP file to install the one binary that's provided in the binary release:
```bash
wget https://releases.hashicorp.com/vault/0.8.3/vault_0.8.3_linux_amd64.zip
```
or go to [HashiCorp's Vault](https://www.vaultproject.io/downloads.html) download page for the latest version if so desired.

#### Installation of the binary
Next step is to unzip the downloaded file and installing it into the existing PATH of the system
(adjust for a different version that you might have downloaded)
```bash
unzip vault_0.8.3_linux_amd64.zip
```
Ensure that the vault file is executable
```bash
chmod +x vault
```
Move the binary `vault` into `/usr/bin`
```bash
mv vault /usr/bin/
```

### Configuration
Create a directory for `vault's` configuration
```bash
mkdir -p /opt/vault/config
```

Create a configuration file for `vault` /opt/vault/conf/vault_config.hcl
```
storage "consul" {
  address = "127.0.0.1:8500"
    path = "vault"
    }

listener "tcp" {
 address = "10.1.2.3:8200"
 tls_disable = 1
}
```

### Note
The listener IP address for vault needs to be an IP address that is accessible on your network for other servers.
Also, the IP address 10.1.2.3 is for demonstration purposes only and needs to be replaced by the actual IP address
of the `vault` server. Furthermore, ensure that the port `8200` is accessible.

## Starting up the servers
### Consul
This script **start_consul.sh** is to start consul, which is used by vault as a key/value store for its secrets.
```bash
#!/usr/bin/env bash

DATA_DIR="/opt/consuldata"

pid=$(ps aux | grep "consul agent" | grep -v grep | awk '{print $2}')
if [ -z ${pid} ] ; then
    nohup consul agent -server -bootstrap-expect 1 -data-dir /opt/consuldata -bind 127.0.0.1 &
else
    echo "Consul Agent is already running as pid ${pid}"
fi
```
Ensure that the script is executable:
```bash
chmod +x start_consul.sh
```
Usage:
```bash
./start_consul.sh
```

### Vault
This script **start_vault.sh** is used to start up the `vault` itself.
```bash
#!/usr/bin/env bash

CONFIG_DIR="/opt/vault/conf"

pid=$(ps aux | grep "vault server" | grep -v grep | awk '{print $2}')
if [ -z ${pid} ] ; then
    nohup vault server -config=$CONFIG_DIR/vault_config.hcl &
else
    echo "Vault Server is already running at pid ${pid}"
fi
```
Ensure that the script is executable
```bash
chmod +x start_vault.sh
```
Usage:
```
./start_vault.sh
```
#### Verification
After the successful startup of `consul` and `vault`, one can verify that these services are
started and ready for use:
```bash
ps aux | grep -e consul -e vault
```

Expect to see something similar to this:
```
root      1744  0.2  2.7  70384 28004 pts/0    Sl   14:13   0:00 consul agent -server -bootstrap-expect 1 -data-dir /opt/consuldata -bind 127.0.0.1
root      1751  0.0  5.6  58464 57008 pts/0    SLl  14:13   0:00 vault server -config=/opt/vault/conf/vault_config.hcl
```

### Unsealing the Vault
The next steps are to unseal the `vault`; by default, it requires three different keys to unseal the `vault`.
Typically, this is done by three different people and each person holds a different key.
The system initializes five different keys upon first initialization.

```bash
vault init
```
This command will produce a key listing like this:
```
Key 1: 427cd2c310be3b84fe69372e683a790e01
Key 2: 0e2b8f3555b42a232f7ace6fe0e68eaf02
Key 3: 37837e5559b322d0585a6e411614695403
Key 4: 8dd72fd7d1af254de5f82d1270fd87ab04
Key 5: b47fdeb7dda82dbe92d88d3c860f605005
Initial Root Token: eaf5cc32-b48f-7785-5c94-90b5ce300e9b

Vault initialized with 5 keys and a key threshold of 3!
```
*The resulting keys need to be kept in a safe location.*

### Enabling Vault
The action of unsealing the `vault` must be performed prior to using the `vault`.
```bash
vault unseal
```
The command is waiting for a key that will be entered hidden like a UN*X password with no echo to the screen.
After the minimum of required keys have been provided successfully, `vault` is ready for action.


## SSH - Secret Backend Setup
### Note
For this POC I used an AWS flavored Ubuntu LINUX AMI (ami-cd0f5cb6, t2.micro instance type).
For this step make sure that both, `consul` and `vault`, are started up and that `vault` is unsealed.

### Mounting the secret backend
```bash
vault mount ssh
```
#### Create a Role for vault's ssh access
Now configure `vault's` role for a OneTimePassword (otp):
```bash
vault write ssh/roles/otp_key_role key_type=otp default_user=ubuntu cidr_list=x.x.x.x/y,m.m.m.m/n
```
The used CIDR list depends on your network setup.

#### Create credentials for vault's ssh access
```bash
vault write ssh/creds/otp_key_role ip=10.11.12.13
```

### SSH client setup
#### Download the SSH-helper
```
wget https://releases.hashicorp.com/vault-ssh-helper/0.1.3/vault-ssh-helper_0.1.3_linux_amd64.zip
```
or you can download the latest version of vault-ssh-helper at [releases.hashicorp.com](https://releases.hashicorp.com/vault-ssh-helper).

##### Unzip the downloaded file
```bash
unzip vault-ssh-helper_0.1.3_linux_amd64.zip
```
##### Ensure that the file is executable
```bash
chmod +x vault-ssh-helper
```
###### Move the executable into a location that's in your PATH
```bash
mv vault-ssh-helper /usr/bin/
```
#### Configuration for vault-ssh-helper
The vault-ssh-helper is like a PAM service by nature.
It accepts an OneTimePassword (otp) provided by `vault` when setup.

*All nodes that shall be reachable by `vault's` OneTimePassword mechanism must have this agent installed*

##### Configuration
Create a directory for the configuration
```bash
mkdir /etc/vault-ssh-helper
```
Then create the configuration file `/etc/vault-ssh-helper/config.hcl`
```
vault_addr = "http://<DNS name for 10.1.2.3>:8200"
ssh_mount_point = "ssh"
tls_skip_verify = true
allowed_roles = "*"
allowed_cidr_list = "10.1.0.0/16"
```
#### Note
The IP address and the network is for demonstration purposes only.
Adjust to your environment.
Furthermore, note that the *vault_addr* should be the DNS name
for the vault server.

##### PAM configuration
Modify /etc/pam.d/sshd file as follows *(put these lines in the beginning of the configuration)*
```
# @include common-auth
auth requisite pam_exec.so quiet expose_authtok log=/tmp/vaultssh.log /usr/bin/vault-ssh-helper -dev -config=/etc/vault-ssh-helper/config.hcl
auth optional pam_unix.so not_set_pass use_first_pass nodelay
```
#### Notes
If there is a directive *@include common-auth* in the /etc/pam.d/sshd configuration file, comment it out.
The setup for vault in production must be done with a valid (not selfsigned) SSL certificate to be secure!
Development mode was chosen here to make the POC work without buying a SSL certificate.

##### SSHD configuration
Modify /etc/ssh/sshd_config as follows:
```
ChallengeResponseAuthentication yes
UsePAM yes
PasswordAuthentication no
```
#### Note
These variables are most likely already configured in the configuration file.
Adjust their values so that they match the above example.

#### Restart SSH
Restart the sshd daemon to re-read the altered configuration

#### Verification
Run the vault-ssh-helper in verify-mode to ensure your configuration and setup is functional.
```bash
vault-ssh-helper -verify-only -config=/etc/vaul-ssh-helper/config.hcl
```
#### SSH keys
Please install the public key of the `vault` server onto the client as usual.

### Client setup is done
This concludes the setup of the vault-ssh-helper on the client.

## Using SSH with vault
### Connect to a remote server using vault from a different machine.
vault ssh -role otp_key_role ubuntu@10.11.12.13

#### Note
For automatic logins install `sshpass` on your system

# MySQL setup
MySQL will need to be setup with a `super` user configured that can create and grant user's access as Vault needs
superuser access itself [See Notes](#footnotes).

Log into your MySQL installation and perform this step:
```
GRANT SUPER ON *.* TO mysqluser@localhost
```

## Mounting the database backend
###Mounting the `database` plugin for Vault
```bash
vault mount database
```

### Configuring vault for use with MySQL
```bash
vault write database/config/mysql \
    plugin_name=mysql-database-plugin \
    connection_url="root:mysqlpassword@tcp(10.20.30.40:3306)/" \
    allowed_roles="readonly"
```
Use the username and password of the user that has all the **SUPER** powers
otherwise this step will fail

#### Notes
*When you have special characters in the password, they need to be URL-encoded*.

### Now, creating a role for Vault to act upon the database
```bash
vault write database/roles/readonly \
    db_name=mysql \
    creation_statements="CREATE USER '{{name}}'@'%' IDENTIFIED BY '{{password}}';GRANT SELECT ON *.* TO '{{name}}'@'%';" \
    default_ttl="1h" \
    max_ttl="24h"
```
## This concludes the MySQL setup and configuration
### Connecting to the MySQL instance with an OneTimePassword (otp)

#### At this point in time a teporary username and password can be requested from Vault
```bash
vault read database/creds/readonly
```
That will result into something similar to this:
```
Key             Value
---             -----
lease_id        database/creds/readonly/2f6a614c-4aa2-7b19-24b9-ad944a8d4de6
lease_duration  1h0m0s
lease_renewable true
password        8cab931c-d62e-a73d-60d3-5ee85139cd66
username        v-root-e2978cd0-
```
#### Connect to MySQL
```bash
mysql -h 10.20.30.40 -u v-root-e2978cd0- -p
```
Then copy and paste the password when MySQL requests the password.
Keep in mind that there will be no echoing of the password by MySQL while entering the password.

### Footnotes
#### SUPER 
https://dba.stackexchange.com/questions/63404/how-to-grant-super-privilege-to-the-user

#### and its caveats
https://dba.stackexchange.com/questions/124066/is-there-any-danger-in-granting-super-privileges-to-a-user

### Temporary IP addresses:
 * Vault Server: 10.250.80.227
 * SSH Client  : 10.250.74.160
 * MySQL Client: 10.250.80.71

All of these instances are UBUNTU based LINUX distributions and as such the username to be used is 'ubuntu'

**These IP addresses will become invalid when the instances are terminated**
