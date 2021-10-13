# Your Very Own Remote Linux Box


Every time I installed a fresh linux box, I would find myself looking up the 
same series of commands over and over to make the newly spawned machine 
distinct and securely accessible on the network.

Then I7600 IoT Design Lab at CCNY incentivized me to write this tutorial 
~~for academic credit~~.

## 1. Requirements

The instructions below are typeset as I am handling ubuntu 20.04 64bit server 
installation on a raspberry pi, though this tutorial should work with other
devices and linux flavors.

The only requirement is a freshly installed linux box (further referred as 
*remote*) that we are about to boot into and a wired connection to a DHCP 
network, so that we can find our box on the network from the computer we are 
about to ssh from (further referred as *local*). This tutorial leaves the 
process of linux installation up to you.

## 2. First Boot

Our device is powered on, let's map it on the network.

### 2.1 You own the network.

In case you own the network and have access the router/network switch you can
look up the ip address of the device on your NAT. Ubuntu device will be called 
`ubuntu` by default. Keep in mind that the ip address will change when DHCP 
lease time is over, most of the routers have an option to make the ip address
sticky.

### 2.2 The network isn't yours ###

This might get tricky since you can't simply determine the ip of your newly 
installed ubuntu box. First you have to be sure that the network switch you are 
connecting to is DHCP (the ip address can be obtained automatically), you 
could poke the ethernet port with your laptop to test.

Now you'd like to determine the ip address of your box. You could run a 
network scan with `nmap` to discover devices on the subnet.

```shell
nmap -sn 192.168.1.0/24  # if your ip is 192.168.1.x
```

In case `nmap` resolved hostnames (you're lucky), just look for `ubuntu` in 
the list. Otherwise, you might have to unplug your device's ethernet and do 
another scan to see which device disappeared/appeared.

## 3. Changing defaults

Once you have determined the ip, it is time to remotely login via ssh.

```shell
ssh ubuntu@192.168.0.123  # Relace the ip of your device.
```

You will be prompted to enter the default password (it's ubuntu in case you are
working with ubuntu).

Next, we would like to change the device hostname (as it appears on the 
network) and username, which would require log in as a different user. We 
will login as *root*. Alternatively we could create another user account and
then delete the default one. In this tutorial I will use the root account.

Set root password.
```shell
sudo passwd root
```

Open `sshd_config` and allow login as root by setting `PermitRootLogin yes`.
```shell
sudo nano /etc/ssh/sshd_config  # Open sshd_config
```

Restart ssh service.
```shell
service sshd restart
```

Logout.
```shell
exit
```

### 3.1 Change hostname

Login as root.
```shell
ssh root@192.168.0.123  # Relace the ip of your device.
```

Hostname is a string that device uses to self identify on the network, it is 
stored in `/etc/hostname`. If you read the file you can see the default "ubuntu"
in there.

My device is a raspberry pi 3b, and I shall set hostname accordingly (so I can
recognize it from the other devices on my local network).
```shell
echo "raspi3b-alpha" > /etc/hostname
```

### 3.2 Change username

Rename the default ubuntu user.
```shell
usermod -l my_username -d /home/my_username -m ubuntu
```

Reboot.
```shell
reboot
```

## 4. Public key authentication

If you don't have a key pair associated with your local machine, create one.
```shell
cd ~/.ssh  # SSH keys are normally stored in the .ssh directory.
ssh-keygen -t ed25519
```

You will be prompted to enter the key name and protect it with a password 
(up to you). This will generate `your-key.pub` and `your-key`, public and 
private keys accordingly. Public key is the one you will share with remotes, 
private key should stay safe and secret on you local computer.

Share public key with the remote.
```shell
scp your-key.pub my_username@192.168.0.123:~/.ssh/.  # Mind the ip.
```

### 4.1 Secure the remote

Login to the remote.
```shell
ssh my_username@192.168.0.123
```

Authorize `your-key.pub` to be a login credential on the remote.
```shell
cat ~/.ssh/your-key.pub >> ~/.ssh/authorized_keys
```

Configure ssh by finding and setting the following in the `/etc/ssh/sshd_config`
```
PermitRootLogin no  # disable root login
PubkeyAuthentication yes  # enable public key authentication.
PasswordAuthentication no  # disable password authentication.
```

Hint: press ctrl+w and type something to search.
```shell
sudo nano /etc/ssh/sshd_config  # open sshd_config with nano
```

Finally, restart ssh to apply the changes.
```shell
service sshd restart
```

Almost forgot, lock the root user.
```shell
sudo passwd -l root
```

Attempt to log in using public key from your local machine. Notice, flag 
`-i` is pointing to the private key. Attempting to log in without passing the 
public key should now fail. 

```shell
ssh my_username@192.168.0.123 -i ~/.ssh/your-key
```

## 5. Tips and tricks

### 5.1. SSH config

Imagine you would like to manage multiple devices from your local machine.
Remembering all ip addresses specifying path to the appropriate keys can get 
annoying quickly. You could add the following directive to you local 
`~/.ssh/config`.

```
Host raspi3b.local # Name of the entry
    HostName 192.168.0.123  # This can also be a dns name like `my-domain.com`.
    User my_username  # Your username goes here.
    IdentityFile ~/.ssh/your-key # Path to your private key.
```

This way you could just type the following to login.
```shell
ssh raspi3b.local
```

In case you're wondering the `.local` is just my way of indication that I would
like to login via the local network. You could set the name to anything.

> **Help Me Improve**  
> I am learning to write meaningful documentation. I hope you enjoyed this post, please help me back by emailing some feedback!
> - Is information clear, correct and up to date?
> - How would you improve this post?
