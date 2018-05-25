==========================
How does in6_pktinfo work?
==========================

:date: 2018-05-25
:tags: netcat, pktinfo, PKTINFO
:category: tech
:summary: How does in6_pktinfo work?

How does in6_pktinfo work?
==========================

The following code used to work just fine with RHEL7.3 kernels and
older. I will demonstrate that this code works in RHEL6.7 (a VM that I
have on my laptop) and fails in CentOS7.5 (another VM). What I found is
that RHEL7.3 and older kernels only give in_pktinfo (never in6_pktinfo).
They essentially worked as IPv4 sockets for this control message. Note
that we do enable both "IPPROTO_IP, IP_PKTINFO" and "IPPROTO_IPV6,
IPV6_RECVPKTINFO" on the socket. If we enable only IPV6_RECVPKTINFO,
RHEL6.7 fails to provide any PKTINFO control data at all. :-(

RHEL7.4 and later kernels provide both in_pktinfo and in6_pktinfo, but
don't work if we use in6_pktinfo while sending the response! There must
be something in this program that I am doing wrong with in6_pktinfo?

Here is the source of the pktinfo server code in pktinfo.c::

    #define _GNU_SOURCE
    #include <netinet/in.h>
    #include <sys/types.h>
    #include <sys/socket.h>
    #include <string.h>
    #include <stdio.h>
    #include <unistd.h>
    #include <stdlib.h>
    #include <stdbool.h>

    /* All globals we need here except socket fd !! */
    struct store {
        struct sockaddr_in6 from;
        struct in_pktinfo in_pktinfo;
        struct in6_pktinfo in6_pktinfo;
        bool pktinfo4;
        bool pktinfo6;
    } store;
    int fd; /* Socket fd */

    static void create_socket(int port);
    static void receive();
    static void respond();
    int main(int argc, char *argv[])
    {
        int port;

        if (argc != 2) {
            fprintf(stderr, "usage: %s <port>\n", argv[0]);
            exit(1);
        }

        port = strtol(argv[1], NULL, 10);
        create_socket(port);
        for (;;) {
            receive(fd, &store);
            respond(fd, &store);
        }
    }

    static void create_socket(int port)
    {
        int enable = 1;
        struct sockaddr_in6 sin6 = {0};
        int ret;

        fd = socket(AF_INET6, SOCK_DGRAM, 0);
        if (fd == -1) {
            perror("socket call failed");
            exit(1);
        }

        setsockopt(fd, SOL_SOCKET, SO_REUSEADDR, &enable, sizeof(enable));
        setsockopt(fd, IPPROTO_IP, IP_PKTINFO, &enable, sizeof(enable));
        setsockopt(fd, IPPROTO_IPV6, IPV6_RECVPKTINFO, &enable, sizeof(enable));

        sin6.sin6_family = AF_INET6;
        sin6.sin6_port = htons(port);
        ret = bind(fd, (struct sockaddr*)&sin6, sizeof(sin6));
        if (ret == -1) {
            perror("bind failed");
            exit(1);
        }
    }

    static void receive(int fd, struct store *store)
    {
        struct iovec iovec[1];
        struct msghdr msg;
        char msg_control[1024];
        char buf[1500];
        struct cmsghdr* cmsg;
        int ret;

        iovec[0].iov_base = buf;
        iovec[0].iov_len = sizeof(buf);
        msg.msg_iov = iovec;
        msg.msg_iovlen = 1;
        msg.msg_name = &store->from;
        msg.msg_namelen = sizeof(store->from);
        msg.msg_control = msg_control;
        msg.msg_controllen = sizeof(msg_control);
        msg.msg_flags = 0;

        ret = recvmsg(fd, &msg, 0);
        if (ret == -1) {
            perror("recvmsg failed");
            exit(1);
        }
        printf("received bytes len: %d\n", ret);

        store->pktinfo4 = false;
        store->pktinfo6 = false;
        for (cmsg = CMSG_FIRSTHDR(&msg); cmsg != 0; cmsg = CMSG_NXTHDR(&msg, cmsg))
        {
            if (cmsg->cmsg_level == IPPROTO_IP && cmsg->cmsg_type == IP_PKTINFO)
            {
                store->in_pktinfo = *(struct in_pktinfo*)CMSG_DATA(cmsg);
                store->pktinfo4 = true;
                printf("pktinfo4 True\n");
            }
            if (cmsg->cmsg_level == IPPROTO_IPV6 && cmsg->cmsg_type == IPV6_PKTINFO)
            {
                store->in6_pktinfo = *(struct in6_pktinfo*)CMSG_DATA(cmsg);
                store->pktinfo6 = true;
                printf("pktinfo6 True\n");
            }
        }
    }

    static void respond(int fd, struct store *store)
    {
        char *resp = "This is PKT6INFO test server\n";
        struct iovec iovec[1];
        struct msghdr msg;
        char msg_control[1024];
        struct cmsghdr *cmsg;
        int control_len;
        int ret;

        iovec[0].iov_base = resp;
        iovec[0].iov_len = strlen(resp);
        msg.msg_name = &store->from;
        msg.msg_namelen = sizeof(store->from);
        msg.msg_iov = iovec;
        msg.msg_iovlen = 1;
        msg.msg_control = msg_control;
        msg.msg_controllen = sizeof(msg_control);
        msg.msg_flags = 0;

        control_len = 0;
        cmsg = CMSG_FIRSTHDR(&msg);
        if (store->pktinfo6)
        {
            cmsg->cmsg_level = IPPROTO_IPV6;
            cmsg->cmsg_type = IPV6_PKTINFO;
            *(struct in6_pktinfo*)CMSG_DATA(cmsg) = store->in6_pktinfo;
            cmsg->cmsg_len = CMSG_LEN(sizeof(store->in6_pktinfo));
            control_len = CMSG_SPACE(sizeof(store->in6_pktinfo));
            printf("Using in6_pktinfo\n");
        } else if (store->pktinfo4) {
            cmsg->cmsg_level = IPPROTO_IP;
            cmsg->cmsg_type = IP_PKTINFO;
            *(struct in_pktinfo*)CMSG_DATA(cmsg) = store->in_pktinfo;
            cmsg->cmsg_len = CMSG_LEN(sizeof(store->in_pktinfo));
            control_len = CMSG_SPACE(sizeof(store->in_pktinfo));
            printf("Using in_pktinfo\n");
        }
        msg.msg_controllen = control_len;

        ret = sendmsg(fd, &msg, 0);
        if (ret == -1) {
            perror("sendmsg failed");
            exit(1);
        }
    }

All the above code does is, it reads UDP packets over the given port, and then
responds with "This is PKT6INFO test server" message. We will use netcat to act
as our client!

Here are the steps to run the above server and see it working on RHEL6.7
-------------------------------------------------------------------------

#. vm1 is running RHEL6.7, let us copy the code there and compile
   pktinfo server::

    $ ssh root@vm1 uname -a
    Linux vm1 2.6.32-573.7.1.el6.x86_64 #1 SMP Thu Sep 10 13:42:16 EDT 2015 x86_64 x86_64 x86_64 GNU/Linux

    scp pktinfo.c root@vm1:
    ssh root@vm1 "make pktinfo"

#. vm1's ip address is 192.168.122.31, I also created a bunch of IP aliases as you can see here::
    
    [root@vm1 ~]# ip addr show dev eth0
    2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1300 qdisc pfifo_fast state UP qlen 1000
        link/ether 52:54:00:07:5c:31 brd ff:ff:ff:ff:ff:ff
        inet 192.168.122.31/24 brd 192.168.122.255 scope global eth0
        inet 192.168.122.200/24 brd 192.168.122.255 scope global secondary eth0:0
        inet 192.168.122.201/24 brd 192.168.122.255 scope global secondary eth0:1
        inet 192.168.122.202/24 brd 192.168.122.255 scope global secondary eth0:2

#. Run our pktinfo server on vm1. Run it in foreground to see its output::

    [root@vm1 ~]# /root/pktinfo 8888

#. Run tshark to see the frames while we run our netcat client::

    tshark -i eth0 -w /tmp/pktinfo.cap port 8888

#. Finally, our netcat client in UDP (we pass STDIN to netcat so that it
   prints response from our pktinfo server!, control-C to terminate
   netcat!)::

    [root@vm2 ~]# cat <(echo command) - | nc -u 192.168.122.202 8888
    This is PKT6INFO test server

#. Note that our server prints the following on its stdout::

    received bytes len: 8
    pktinfo4 True
    Using in_pktinfo

If you run the pktinfo server on RHEL7.[45], your netcat won't print our
pktinfo server response message! Network trace will indicate that our
server sends a response to netcat using a different source IP address
than the destination address of the netcat message. Also, notice that
the output from the pktinfo server is::

    received bytes len: 8
    pktinfo6 True
    pktinfo4 True
    Using in6_pktinfo

The kernel is sending IP_PKTINFO as well as IPV6_PKTINFO, it works only
when we use IP_PKTINFO while sending a UDP packet (I did re-arrange the
pktinfo.c code to use in_pktinfo first, then things work in RHEL7.4
kernel as well. Looks like there is a bug in this code or in the kernel.
