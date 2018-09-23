# Introduction
Welcome to the hopefully painless GNUnet tutorial for Ubuntu 18.04! It provides very concrete instructions on how to compile, install and configure a current version of GNUnet. The goal is to support newcomers, either end users or developers, who want to get in touch with GNUnet for the first time. After installing GNUnet we will make sure that out new GNUnet installation is working correctly.

**Attention: If you came across the official gnunet package for Ubuntu 18.04, ignore it! It is ancient and not compatible with current GNUnet installations.**

Now let's start!

# Requirements
First let's install the following Ubuntu 18.04 packages to use GNUnet painlessly. Optional dependencies are listed in Appendix A. They are required for some experimental GNUnet features.

```
$ sudo apt install libtool autoconf autopoint build-essential libgcrypt-dev libidn11-dev zlib1g-dev libunistring-dev libglpk-dev miniupnpc libextractor-dev libjansson-dev libcurl4-gnutls-dev libsqlite3-dev libmicrohttpd-dev
```

# Make an installation directory
Next we create a directory in our home directory where we store the source code later. We should keep this directory after installation because it contains Makefiles that can be used for uninstalling GNUnet again (see chapter *Uninstall GNUnet and its dependencies*).

```
$ mkdir ~/gnunet_installation
```

# Get the source code
We download the GNUnet source code using git.

```
$ cd ~/gnunet_installation
$ git clone --depth 1 https://gnunet.org/git/gnunet.git
```

# Compile and Install
Installing GNUnet is not hard, it only requires one little nasty step which involves modifying an important config file of the operating system. So we'll pay extra attention while doing this.

We have two options: installing a *production version* and installing a *development version*. If you want to start writing GNUnet applications or join the GNUnet development choose the development version (it will print more debug output and contains debug symbols that can be displayed with a debugger). Otherwise choose the production version.
 
## Option 1: GNUnet for production / usage
```
$ cd ~/gnunet_installation/gnunet
$ ./bootstrap
$ export GNUNET_PREFIX=/usr
$ ./configure --prefix=$GNUNET_PREFIX --disable-documentation
$ sudo addgroup gnunetdns
$ sudo adduser --system --group --disabled-login --home /var/lib/gnunet gnunet
$ make -j$(nproc || echo -n 1)
$ sudo make install
```

## Option 2: GNUnet for development
```
$ cd ~/gnunet_installation/gnunet
$ ./bootstrap
$ export GNUNET_PREFIX=/usr
$ export CFLAGS="-g -Wall -O0"
$ ./configure --prefix=$GNUNET_PREFIX --disable-documentation --enable-logging=verbose
$ make -j$(nproc || echo -n 1)
$ sudo make install
```

## Install GNUnet plugin for name resolution
So now it gets a bit nasty. It's not so bad. All we have to do is copy a file and edit another one. The file we need to copy is GNUnet's plugin for the Name Service Switch (NSS) in unix systems. Different unixes expect it in different locations and GNUnet's build system does not try to guess. On Ubuntu 18.04 we have to do

```
$ sudo cp /usr/lib/gnunet/nss/libnss_gns.so.2 /lib/$(uname -m)-linux-gnu/
```

The next step is activating the GNUnet plugin we just copied in the NSS config. It is located in `/etc/nsswitch.conf`. It should contain a line starting with "hosts" similar to this (at least "files" and "dns" should be there):

```
$ cat /etc/nsswitch.conf
hosts: files mdns4_minimal [NOTFOUND=return] dns
```

**Attention: Once we modified `etc/nsswitch.conf` DNS resolution will only be possible as long as is GNUnet is running. We can leave the next step out, but then we will not be able to use GNUnet's name resolution in external applications.**

We save a copy of the original file and then modify the line using sed:

```
$ sudo cp /etc/nsswitch.conf /etc/nsswitch.conf.original
$ sudo sed -i -E 's/^(hosts:.*) dns/\1 gns [NOTFOUND=return] dns/' /etc/nsswitch.conf 
```

Now in the line starting with "hosts" should contain an entry "gns [NOTFOUND=return]" before the "dns" entry like this:

```
hosts: files mdns4_minimal [NOTFOUND=return] gns [NOTFOUND=return] dns
```

That's it. It wasn't that nasty, was it?

# Configuration
Congratulations! GNUnet is now installed! Before we start it we need to create a configuration file. By default GNUnet looks in our home directory for the file `~/.gnunet/gnunet.conf`. We can start with an empty file for now:

```
$ touch ~/.config/gnunet.conf
```

Now we can start it with the command line tool `gnunet-arm` (Automatic Restart Manager).

```
$ gnunet-arm -s
```

It starts the default GNUnet services. We can list them with the `-I` option:

```
$ gnunet-arm -I
Running services:
ats (gnunet-service-ats)
revocation (gnunet-service-revocation)
set (gnunet-service-set)
nat (gnunet-service-nat)
transport (gnunet-service-transport)
peerstore (gnunet-service-peerstore)
hostlist (gnunet-daemon-hostlist)
identity (gnunet-service-identity)
namecache (gnunet-service-namecache)
peerinfo (gnunet-service-peerinfo)
datastore (gnunet-service-datastore)
zonemaster (gnunet-service-zonemaster)
zonemaster-monitor (gnunet-service-zonemaster-monitor)
nse (gnunet-service-nse)
cadet (gnunet-service-cadet)
dht (gnunet-service-dht)
core (gnunet-service-core)
gns (gnunet-service-gns)
statistics (gnunet-service-statistics)
topology (gnunet-daemon-topology)
fs (gnunet-service-fs)
namestore (gnunet-service-namestore)
vpn (gnunet-service-vpn)
```

For stopping GNUnet again we can use the `-e` option.

```
$ gnunet-arm -e
```

# Make sure it works
Let's try some of GNUnet's components: gns, filesharing, CADET and VPN.

## GNS
First let's try out GNS, the GNU name service. We'll publish an IP address in a GNS record and try to resolve it using our browser. First we need an identity which is the equivalent to a zone in DNS. We'll call it "myself" and create it using the `gnunet-identity` command line tool. 
Instead of "myself" you can surely use your nick or any other name. 

```
$ gnunet-identity -C myself
```

We can check if it worked using the same tool. We expect the name of our identity and the corresponding public key to be displayed.

```
$ gnunet-identity -d
myself - HWTYD3P5D77JVFNVMZ1M5T10V4SZYNMY3PCGQCSVENKD6ZCRKPMG
```

Now we add a public `A` record to our zone. It has the name "ccc", a value of "195.54.164.39" and it never expires.
```
$ gnunet-namestore -z myself -a -e never -p -t A -n ccc -V 195.54.164.39
```

Now we can query that record using the command line tool `gnunet-gns`.

```
$ gnunet-gns -u ccc.myself
ccc.myself:
Got `A' record: 195.54.164.39
```

So it worked! Now you can try to type "ccc.myself" into your browser and see what website is behind the IP address. (If it doesnt work use the IP directly ;p)

## filesharing
Let's publish a file in the GNUnet filesharing network. We use tow keywords ("commons" and "state") so other people will be able to search for the file. 

We can choose any file and describe it with meaningful keywords (using the `-k` command line option).

```
$ gnunet-publish -k commons -k state ostrom.pdf
Publishing `/home/myself/ostrom.pdf' done.
URI is `gnunet://fs/chk/M57SXDJ72EWS25CT6307KKJ8K0GCNSPTAZ649NA1NS10MJB4A1GZ9EN4Y02KST9VA5BHE8B335RPXQVBWVZ587Y83WQ7J3DHMBX30Q8.DHNGBN4CB2DBX1QRZ1R0B1Q18WTEAK4R94S9D57C9JMJJ3H7SSQDCV4D1218C4S2VP085AMQQSMG18FCP6NQMZQZJ91XR5NBX7YF0V0.42197237'.
```

Finding the file by keyword works with `gnunet-search`.

```
$ gnunet-search commons
#1:
gnunet-download -o "ostrom.pdf" gnunet://fs/chk/M57SXDJ72EWS25CT6307KKJ8K0GCNSPTAZ649NA1NS10MJB4A1GZ9EN4Y02KST9VA5BHE8B335RPXQVBWVZ587Y83WQ7J3DHMBX30Q8.DHNGBN4CB2DBX1QRZ1R0B1Q18WTEAK4R94S9D57C9JMJJ3H7SSQDCV4D1218C4S2VP085AMQQSMG18FCP6NQMZQZJ91XR5NBX7YF0V0.42197237
```

It gives us the command line call to download the file (and store it as ostrom.pdf)!


## CADET (and Chat)

We can use the `gnunet-cadet` command line tool to open a port and from another machine connect to this port and chat or transfer data. First we need our *peer ID* of the GNUnet peer opening the port.

```
$ gnunet-peerinfo -s
I am peer `P4T5GHS1PCZ06R82D3KW8Z8J1113BQZWAWGYHTZ8G1ZXMWXQGAVG'.
```

Now we open the port (it can be any string!):

```
$ gnunet-cadet -o my-secret-port
```

On the other machine we can connect using the peer ID and the port and start chatting!

```
$ gnunet-cadet P4T5GHS1PCZ06R82D3KW8Z8J1113BQZWAWGYHTZ8G1ZXMWXQGAVG my-secret-port
```

## VPN
TBD

# Uninstall GNUnet and its dependencies
```
$ cd ~/gnunet_installation/gnunet
$ sudo make uninstall
$ sudo apt remove libtool autoconf autopoint build-essential libgcrypt-dev libidn11-dev zlib1g-dev libunistring-dev libglpk-dev miniupnpc libextractor-dev libjansson-dev libcurl4-gnutls-dev libsqlite3-dev libmicrohttpd-dev
$ sudo apt autoremove
$ sudo userdel -r gnunet
$ sudo groupdel gnunet
$ sudo groupdel gnunetdns
$ sudo mv /etc/nsswitch.conf.original /etc/nsswitch.conf
$ sudo rm /lib/$(uname -m)-linux-gnu/libnss_gns.so.2
```


# Appendix A: Optional GNUnet features
TBD

# Troubleshooting

## You can't reach other people's nodes

Should our computer not have reached the open GNUnet network automatically, we can manually instruct our node how to reach the nodes of our friends. This works by exchanging HELLO strings. This is how we get a hello string for our computer.

```
$ gnunet-peerinfo -gn
```

We can now pass this string to our friends "out of band" (using whatever existing chat or messaging technology). If the string contains some private IP networks we don't want to share, we can carefully edit them out.

Once we receive such strings from our friends, we can add them like this:

```
 gnunet-peerinfo -p <string>
```
    
Now our GNUnet nodes can attempt reaching each other directly. This may still fail due to NAT traversal issues.


## OMG you guys broke my internet

We can replace `/etc/nsswitch.conf` with the backup we made earlier (`/etc/nsswitch.conf.original`). Now DNS resolution should work again without a running GNUnet.

```
$ cp /etc/nsswitch.conf.original /etc/nsswitch.conf
```
