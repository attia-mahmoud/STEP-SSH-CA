# STEP-SSH-CA
This guide provides step-by-step instructions on how to use Step to create a private certificate authority (CA) and sign and issue SSH certificates to allow mutual connections between worker and master nodes. Please follow the instructions below:


## Certificate Authority Setup
1. Download and install the **Step CLI** by running the following commands:
   
```
wget https://dl.smallstep.com/gh-release/cli/docs-cli-install/v0.23.4/step-cli_0.23.4_amd64.deb
dpkg -i step-cli_0.23.4_amd64.deb
```

2. Download and install the **Step CA** by executing the following commands:
   
```
wget https://dl.smallstep.com/gh-release/certificates/docs-ca-install/v0.23.2/step-ca_0.23.2_amd64.deb
dpkg -i step-ca_0.23.2_amd64.deb
```

3. Initialize the Step CA with **SSH support enabled** by running the command:

```
step ca init --ssh
```

You should get an output similar to below:

```
Generating root certificate... done!
Generating intermediate certificate... done!
Generating user and host SSH certificate signing keys... done!

✔ Root certificate: /root/.step/certs/root_ca.crt
✔ Root private key: /root/.step/secrets/root_ca_key
✔ Root fingerprint: 9050692596d69c12b061210e5f5fc5daf4684377de887af5ab3f7459a1fe9381
✔ Intermediate certificate: /root/.step/certs/intermediate_ca.crt
✔ Intermediate private key: /root/.step/secrets/intermediate_ca_key
✔ SSH user public key: /root/.step/certs/ssh_user_ca_key.pub
✔ SSH user private key: /root/.step/secrets/ssh_user_ca_key
✔ SSH host public key: /root/.step/certs/ssh_host_ca_key.pub
✔ SSH host private key: /root/.step/secrets/ssh_host_ca_key
✔ Database folder: /root/.step/db
✔ Templates folder: /root/.step/templates
✔ Default configuration: /root/.step/config/defaults.json
✔ Certificate Authority configuration: /root/.step/config/ca.json
```
4. **Start the Step CA server** by running:

```
step-ca $(step path)/config/ca.json
```

## Master Setup
1. **Bootstrap your CA configuration** on your master node by running:

```
step ca bootstrap --ca-url [CA URL] --fingerprint [CA fingerprint]
```

You should get an output similar to below:
```
The root certificate has been saved in /home/alice/.step/certs/root_ca.crt.
Your configuration has been saved in /home/alice/.step/config/defaults.json.
```

The CA URL and CA fingerprint can be found in the defaults.json file on the CA server.

```
cat /root/.step/config/defaults.json
```

Or alternatively to get the CA fingerprint, run the following command on the CA:

```
step certificate fingerprint $(step path)/certs/root_ca.crt
```

2. Add the **SSH User Public Key** to the master by running:

```
step ssh config --roots > /path/to/ssh_user_key.pub
```

3. Add this key to the **master's SSHD configuration** by running:

```
$ cat <<EOF >> /etc/ssh/sshd_config
> # This is the CA's public key for authenticating user certificates:
> TrustedUserCAKeys /path/to/ssh_user_key.pub
> EOF
```

4. **Restart the SSH server.** Your host will now trust any user certificate issued by the CA.
```
service ssh restart
```

## Worker Setup
This will issue an SSH user certificate.

1. **Bootstrap your CA configuration** on your worker node by running the same as Step 1 above.

2. Create an **SSH user certificate** for the user `root` by running:
```
step ssh certificate worker1@nopasaranhosts-worker id_ecdsa
```
You will get an output similar to below:
```
$ step ssh certificate worker1@nopasaranhosts-worker id_ecdsa
✔ Provisioner: carl@smallstep.com (JWK) [kid: yWa7WGfoSt9yJ0OZCndrvR_m65jzDriY7mhPz094fdw]
✔ Please enter the password to decrypt the provisioner key:
✔ CA: https://127.0.0.1:4337
Please enter the password to encrypt the private key:
✔ Private Key: id_ecdsa
✔ Public Key: id_ecdsa.pub
✔ Certificate: id_ecdsa-cert.pub
✔ SSH Agent: yes
```
3. If the step above was unable to **add the private key to the SSH agent**, you can do it manually by running:
```
eval "$(ssh-agent -s)"
ssh-add /path/to/private-key
```

Now you have a certificate for your worker node, try to connect to the master node.
