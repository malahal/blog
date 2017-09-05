=========
technotes
=========

:date: 2017-09-07
:tags: rsync, netcat, tcpdump, iptables
:category: technotes
:summary: Technical cheat sheet for my own consumption!

All in a single page for now!

iptables
=========

port forwarding
----------------

If you have a virtual machine running NAT'ed default libvirt network and
want to host a service (e.g. NFSv4 server) using your host's IP address,
you need to do the following:

- Make sure your host can forward packets (aka act as a gateway)::

      echo 1 | sudo tee /proc/sys/net/ipv4/ip_forward

- Disable firewall. You don't have to, if you now what you are doing! ::

      sudo iptables -F

- Forward tcp NFSv4 packets (aka destination port 2049) to your VM's IP address
  (assume the host ip address is 192.168.1.13 at network interface eth0 and
  your VM's ipaddress is 192.168.122.32)::

     sudo iptables -t nat -A PREROUTING -i <eth0> -p tcp --dport 2049 -j DNAT --to-destination 192.168.122.32 

Now you can mount from a system in your host's network provided your
NFSv4 server is configured to allow it::

    mount -overs=4 192.168.1.13:/gpfs/gpfsB /mnt

You need to forward portmapper port 111 and other side-band protocol ports to
make NFSv3 work. It is left as an exercise to the reader! 

netcat
======

Test your network speed using netcat
-------------------------------------

On one node, start netcat in listening mode::

    nc -v -l -p 2222 > /dev/null # -p may not be needed in some versions

On the other node, send the data as below::

    pv /dev/zero | nc -v <ip-address-of-node-one> 2222

ssh
====

ssh-copy-id with a non default port ::

    ssh-copy-id "user@host -p 1234"

ssh execute arbitrary script on a remote machine! ::

    ssh user@remotehost "echo `base64 script.sh` | base64 -d | sudo
    bash"

Rsync
=====

pulling only few files
-----------------------

Rsync is a nice file synchroniser. If you want to pull only few files,
the syntax is a bit confusing though. The following would pull the
matched files and all directory entries. ::

    rsync -av --include '*.jpg' --include '*/' --exclude '*' host:src dst

It will not pull any jpeg files under subdirectories if you don't
include the second --include! Of course, you can use _find_ to prune the
empty directories as below.::

    find . -type d -empty -delete

Copying a very large file over a slow link
------------------------------------------

It is probably easier to use __lftp__ to do this but not all sites provide
ftp service. __ssh__ is ubiquitous, so you can use rsync as below::

    rsync -a --partial --progress host:src dst

If you need to restart::

    rsync -a --partial --append --progress host:src dst

Rsync with non-standard ssh port
--------------------------------

Example with port 1602::

    rsync -e 'ssh -p 1602' user@dest:/path/to/files /local/path

Running rsync with an intermediate hop!
----------------------------------------

See the link http://rsync.samba.org/firewall.html for now.


tcpdump
=======

Some examples to capture network packets::

    tcpdump -i any -s0 -C 500 -W 20 -w /tmp/pcap.out
    tcpdump -i any -s250 -C 500 -W 20 -w /tmp/pcap.out port nfs
    tcpdump -i any -s250 -C 500 -W 20 -w /tmp/pcap.out port nfs host 192.168.1.122

On some older systems, -C or -W option causes tcpdump to run as tcpdump
user. Use "-Z root" to run it as root.

tshark/wireshark time zone
==========================

Wireshark/tshark use local system time zone. If the capture is done
in a different time zone, set TZ environment to display time in the
captured time zone. Use tzselect for the exact string for the TZ
environment variable. For example::

    TZ='America/Phoenix' capinfos -ae tcpdump.pcap

GNU Parallel and xargs
======================

[GNU parallel][xxx] is written with parallelization in mind. It uses the
number of CPU threads in the system by default. It is not already
installed on most distros, but xargs is, and has this feature as well.
Without -n option, xargs may try to pass all input arguments to a single
command. So always provide -n option with -P if you want to run multiple
commands in parallel. ::

    find . -type f | xargs -P8 -n1 grep my-search-expr
    find . -type f | xargs -I{} -P8 -n1 dd if={} of=/dev/null
    seq 10 | xargs -I{} -P10 -n1 dd if=/tmp/samefile of=/dev/null
    echo 'your list of commands' | xargs -d\n -I{} -P8 -n1 sh -c '{}'

Python
======

Time in python::

    time.ctime()
    datetime.datetime.now()
    strptime and strftime.

    tzpdx = dateutil.tz.tzfile("/usr/share/zoneinfo/America/Los_Angeles")
    time = datetime.datetime.now(tzpdx).strftime("%Y-%m-%d %I:%M%p")

Blog update
===========

Edit files in content directory and then run::

    pelican publish
    ghp-import output
    git push -f git@github.com:malahal/malahal.github.io.git gh-pages:master
