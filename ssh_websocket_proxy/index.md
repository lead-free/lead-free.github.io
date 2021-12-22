# Reverse SSH over Websocket


Sometimes you might find yourself in dark places where ssh is not allowed, and
you must find a way to communicate with the outer world with something other
than HTTP/HTTPS. Why would they do it in the first place, you might be asking.
Sshing into the outer world is 0 steps away from sshing inwards. 
Meaning if there is a way out, there is always a way in. Reverse ssh duh!

Once upon a blue moon, my comrades and I were sent to work from home for
reasons well known, leaving us away from the computational capacities 
~~and free electricity~~. Colleagues were divided into two categories, 
the ones who used remote desktop apps, and the ones who used reverse ssh.
At that point inward ssh was already blocked. We have attempted the civilized 
path as we were pending for the mystical and perhaps non-existing vpn access 
to our organization.

Their firewalls blocked all remote desktop apps, and then some 6 month later all
outgoing ssh connections went down as well (so did reverse ssh).
The curious reader is aware -- regardless of the hardware, service, or 
encoding, connect it to the internet and someone's gonna own it 
(credit to dual core).

## 1. Requirements ##

0. A *captive machine* we want to set free. Open SSH installed, no root 
   permissions required.
1. A machine in the demilitarized zone. Must be always up and running. 
   Could be at your house or in the cloud. Further, referred as free server. 
2. Access to both *free* and *captive* servers to set things up.
3. [Websocat](https://github.com/vi/websocat). Binaries available for unix-like
   x86 systems. Builds painlessly on ARM (build guide in the repo).
4. Open SSH client and server on both *free* and *captive* servers.
5. Open SSL command line tool on the *free server*.

## 2. Free Server ##

#### 2.1 Generating Certificate ####

First, we have to set up a certificate to encrypt the websocket connection.
Firewalls I encountered restrict outgoing ssh connections by sniffing the ssh 
version header, which gets transmitted unencrypted at the initial handshake.
This method won't help you if your websocket communication is plaintext.

Generate the certificate on the free server.
```shell
openssl req -x509 -newkey rsa:4096 -keyout wss-key.pem -out wss-cert.pem -days 365 --nodes
```

In case you have openssl ^v3.0, I would recommend using dsa key.
The command would be.
```shell
openssl req -x509 -newkey dsa -keyout wss-key.pem -out wss-cert.pem -days 365 --noenc
```
Either command would spit out a private key and a certificate. 
Hint: the second 's' stands for *secure*.

Bundle the private key and the certificate into an archive to feed to `websocat`.
```shell
openssl pkcs12 -export -out wss.pkcs12 -inkey wss-key.pem -in wss-cert.pem
```

#### 2.2 Persistent Websocat ####

Time to set up `websocat`. We would like to make our *free server* persistent, 
so that if the machine reboots, the server program comes back online
automagically. Below is a simple shell script to start a `websocat` server on 
port 1337 that decrypts and redirects all traffic into `localhost:22` (aka ssh)
in binary form. In addition, output of `websocat` is logged for further 
debugging if needed.

I will further refer to the following shell script as `wss-server.sh`
```shell
# Content of `wss-server.sh`
 
# Path to websocat executable
websocat=~/websocat
# Path to certificate bundle
pkcs12=~/wss.pkcs12
# Log file (optional)
log=~/websocat.log

echo "Websocat server starting on $(date)" >> $log
$websocat --binary wss-l:127.0.0.1:1337 tcp:127.0.0.1:22 --pkcs12-der $pkcs12 >> $log 2>&1
```

`websocat` itself is quite reliable -- I've been running it for months 
without restarting the process, but your *free server* might not be, as your
roommate reaches to restart the router because they think it could fix their
laggy bloated laptop and by accident your raspberry pi gets unplugged.

It's a good thing raspberry pi boots by itself when the power is back. Let's 
make our websocket server to do the same. This can be achieved by adding
`wss-server.sh` to crontab.

Open crontab file.
```shell
crontab -l
```

Add the following line substituting your path to `wss-server.sh`.
```crontab
@reboot ~/.local/bin/wss-server.sh
```

The `websocat` server will start on boot. It's worth to try running 
`wss-server.sh` from the command line before restarting.

## 3. Captive Server ##

Here I will skip the steps of generating a keypair for securing your ssh 
connection, you can find a step-by-step guide [here](https://stansotn.com/linux_box/#4-public-key-authentication)
(note, this needs to be done for both machines).

First download the latest release of `websocat` for your system from [here](https://github.com/vi/websocat/releases).

```shell
wget url/to/the/download/link
```

Set up ssh proxy. The following to be added into `~/.ssh/config`.
```
host ws.freeserver
    User username
    ProxyCommand ~/websocat --binary wss://<ip or hostname of free server> --insecure
    IdentityFile ~/.ssh/captive-private-key
```

The `--insecure` flag is set here to disable TLS certificate validation with 
trusted certificate authorities. The encryption is used in the websocket layer
of communication to make it non-trivial for the firewall to filter your packets.
Rely on ssh encryption for security. If for whatever reason you want to use 
verifiable certificate, you can get one for free from [let's encrypt](https://letsencrypt.org).
Note, then you would have to use appropriate certificate bundle on the 
*free server*.

Now you can ssh into the *free server* from the *captive server*.
```shell
ssh ws.freeserver
```

If you'd like to leave a backdoor to be able to access the captive server 
from anywhere on the internet you can use a reverse ssh configuration.

The following command mounts port 22 of the *captive server* to port 1338 of 
the *free server*. A good explanation of what ssh forwarding does can be found
[here](https://unix.stackexchange.com/questions/115897/whats-ssh-port-forwarding-and-whats-the-difference-between-ssh-local-and-remot)
```shell
ssh -R 1338:localhost:22 ws.freeserver
```

You can now ssh into the *captive server* as you would normally ssh into the
*free server*, just specify the port.
```shell
# Will get you into the *captive server*
ssh freeserver -p 1338
```

To run the above-mentioned command in the background, add `-N -f` flags. 
SSH connection has a tendency of dropping, use `autossh` to keep the connection
alive (you might have to download the widely available executable). 
Add the following to `crontab` if you'd like to start the ssh tunnel on every
boot.

```crontab
@reboot autossh -NRf 1338:localhost:22 ws.freeserver
```

## 4. Afterword ##

Did you know you could use ssh as poor man's vpn, for instance, you can browse
the internet from anywhere as if you were browsing from the *captive server*, 
see [sshuttle](https://github.com/sshuttle/sshuttle).

In case your ip is blacklisted by the captive's firewall, you can set up dns 
record and proxy for on cloudflare for free.

In case you would like [nginx](https://nginx.org) to proxy connections at the
*free server* with uri path resolution, here is an example configuration to 
proxy websocket connections with nginx.

```nginx.conf
server {

       listen 443 ssl;
       ssl_certificate /path/to/cert.pem;
       ssl_certificate_key /path/to/key.pem;

       location / {
               proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
               proxy_set_header Host $host;

               proxy_pass https://localhost:1337;
               proxy_http_version 1.1;

               proxy_set_header Upgrade $http_upgrade;
               proxy_set_header Connection "Upgrade";

               proxy_read_timeout 1000;
               proxy_ssl_certificate /path/to/cert.pem;
               proxy_ssl_certificate_key /path/to/key.pem;
       }
}
```

> **Help Me Improve**  
> I am learning to write meaningful documentation. I hope you enjoyed this post, please help me back by emailing some feedback!
> - Is information clear, correct and up to date?
> - How would you improve this post?

