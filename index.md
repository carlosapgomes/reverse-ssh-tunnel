# Reverse SSH Tunnel

## References

[Reverse SSH Tunnel](https://hobo.house/2016/06/20/fun-and-profit-with-reverse-ssh-tunnels-and-autossh/)

[How to Create Persistent Reverse SSH Tunne](https://sleeplessbeastie.eu/2014/12/23/how-to-create-persistent-reverse-ssh-tunnel/)

[How to create a restricted SSH user for port forwarding](https://askubuntu.com/questions/48129/how-to-create-a-restricted-ssh-user-for-port-forwarding)


## Sample Use Case

Computer A -> a server behind a firewall that you can not
SSH to directly, but can initiate outbond SSH connections

Computer B -> a cloud virtual machine with a real IP accessible from the
internet that can accept inbound SSH connections

Computer C -> a personal computer with access to the internet from where
you can log in to computer B as an admin

Objective: get an SSH session from computer C to computer A for remote work

You must have root priviledge in both computers A and B.

### Set up computer B

#### Create a dedicated user for the tunnel

`sudo useradd -m -s /sbin/nologin autotunnel`

### Set up computer A

#### Install `autossh`

`sudo apt install autossh`

#### Create a dedicated user for the tunnel to originate the connection

`sudo useradd -m -s /sbin/nologin autotunnel`

#### Create an SSH key pair for this user

Impersonate this new user:

`sudo su - autotunnel -s /bin/bash`

And type the following command:

`ssh-keygen -o -a 100 -t ed25519 -f ~/.ssh/id_autotunnel`

Set an empty passphrase and confirm the other defaults.

Type `exit` to go back to the original logged in user.

#### Copy the public key to computer B

Copy the file `/home/autotunnel/.ssh/id_autotunnel.pub` from computer A to
computer C (from where you can log in as an admin user to computer B). Use
a portable media or send this file as an email attachment to yourself.

From computer C, use `sftp` to transfer that file to your home folder in
computer B.

From the folder where you saved the public key file, start an `sftp` connection
to computer B:

`sftp -i ˜/.ssh/my_vm_id_key username@my.vm.ip.address`

Use `put id_autotunnel.pub` to upload the file.

Close the sftp connection (type `exit`).

`ssh` into computer B.

Create an `.ssh` directory in `autotunnel` home folder:

`sudo su -s /bin/sh autotunnel -c "mkdir ~/.ssh"`

Add the uploaded public key to the `authorized_keys` file:

`echo 'no-agent-forwarding,no-user-rc,no-X11-forwarding,no-pty' $(cat id_autotunnel.pub) | sudo su -s /bin/bash autotunnel -c "tee >> ~/.ssh/authorized_keys"`

#### Add computer B to the known_hosts file

From the console in computer A:

`ssh-keyscan -H -t ed25519 compute.B.ip.address | sudo su -s /bin/sh autotunnel -c "tee >> ~/.ssh/known_hosts"`

### Test the tunnel

#### From computer A

Verify if ssh is running:

`sudo systemctl status ssh`

Open the tunnel with the following command:

`sudo su -s /bin/sh autotunnel -c "autossh -v -i ~/.ssh/id_autotunnel computer.B.ip.address -N -R 9000:localhost:22"`

It will show several lines of information, check if you get something like:

```
...

debug1: Reading configuration data /etc/ssh/ssh_config

...

Authenticated to remote-server ([computer.B.ip.address]:22).

...

debug1: remote forward success for: listen some_port_number, connect 127.0.0.1:some_other_port_number

debug1: remote forward success for: listen 9000, connect localhost:22

debug1: All remote forwarding requests processed

```

The above message says that the tunnel is  working. To interrupt the tunnel
just press `Control-C`.

You can actually test the tunnel from computer B with the following command:

`ssh -o PubkeyAuthentication=no -p 9000 username_on_comp_A@127.0.0.1`

### Make the tunnel persistent

#### From computer A

```
cat > /etc/systemd/system/autossh-myserver.service << EOF

[Unit] 
Description=Keep a tunnel to 'myserver' open 
After=network-online.target


[Service]
Type=forking
User=autotunnel
ExecStart=/usr/bin/autossh -f -M 0 -o ServerAliveInterval=30 -o ServerAliveCountMax=3  -N autotunel@computer.B.ip.address -R 9000:127.0.0.1:22
ExecStop=/usr/bin/pkill -9 -u autotunnel
Restart=always

[Install]
WantedBy=multi-user.target
EOF
```

Enable the service:

`systemctl enable autossh-myserver.service`

Start the service:

`systemctl start autossh-myserver.service`

### Use the tunnel

From computer C, `ssh` into computer B and use the command
bellow to get access to computer A:

`ssh -o PubkeyAuthentication=no -p 9000 username_on_comp_A@127.0.0.1`

