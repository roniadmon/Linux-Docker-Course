## Background matterial

Read [these slides](https://cse.sc.edu/~zeng1/cis3207-spring18/slides/02-cpu-mode.pdf) to understand the difference between kernel space and user space.
Notice that the hardware has its part in this distinction! See [Wikipedia](https://en.wikipedia.org/wiki/Protection_ring) for a bit more on this topic.

To understand namespaces [it's always fun when there's a man page](http://man7.org/linux/man-pages/man7/namespaces.7.html), kinda makes it more official. Also, [Wikipedia](https://en.wikipedia.org/wiki/Linux_namespaces) is very clear and helpful.
If you want to dive deeper, Rami Rozen (Haifa Linux Club) has a good [slide deck](http://www.haifux.org/lectures/299/netLec7.pdf) and [a YouTube lecture](https://www.youtube.com/watch?v=YOOmBHAk6Ls) to go with it. Finally, [this articles series on LWN](https://lwn.net/Articles/531114/) explains namespaces in details.

## Let's get to business

### Goal

In this exercise we're going to set up a DNS (and DHCP, but don't tell anyone) server and client. This will teach us about networking, namespaces and the fun one can have with Linux.

### Steps

1. On your Linux machine, install [`dnsmasq`, a lightweight DNS and DHCP server](http://www.thekelleys.org.uk/dnsmasq/doc.html) which is THE default choice for these usages. Run

   ```sh
   sudo yum install dnsmasq
   ```

   1. You may want to read about [DNS, Domain Name System](https://en.wikipedia.org/wiki/Domain_Name_System), as we'll use this service throught this exercise.
   1. Also, if you want to do the expert extention of this exercise, read on about [DHCP, Dynamic Host Configuration Protocol](https://en.wikipedia.org/wiki/Dynamic_Host_Configuration_Protocol)

1. To get the basics of `dnsmasq`, let's get it running pretty stupidly

   ```sh
   sudo cat > /etc/dnsmasq.conf <<EOF
   address=/i.am.not/127.0.0.3
   user=dnsmasq
   group=dnsmasq
   EOF
   sudo service dnsmasq start
   dig i.am.not a @127.0.0.1
   ```

   you probably got an output kind of like this

   ```plain
   [user@localhost ~]$ service dnsmasq restart
   Redirecting to /bin/systemctl restart dnsmasq.service
   [user@localhost ~]$ dig i.am.not a @127.0.0.1
   ; <<>> DiG 9.11.11-RedHat-9.11.11-1.fc31 <<>> i.am.not a @127.0.0.1
   ;; global options: +cmd
   ;; Got answer:
   ;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 11821
   ;; flags: qr aa rd ra ad; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1

   ;; OPT PSEUDOSECTION:
   ; EDNS: version: 0, flags:; udp: 4096
   ;; QUESTION SECTION:
   ;i.am.not.			IN	A

   ;; ANSWER SECTION:
   i.am.not.		0	IN	A	127.0.0.3

   ;; Query time: 0 msec
   ;; SERVER: 127.0.0.1#53(127.0.0.1)
   ;; WHEN: Thu Dec 19 15:42:36 IST 2019
   ;; MSG SIZE  rcvd: 53
   ```

   * To undestand what is `Redirecting to /bin/systemctl restart dnsmasq.service`, run `man systemd`and `man systemctl`. Also see [SystemD in Wikipedia](https://en.wikipedia.org/wiki/Systemd).
   * Do you undertand the output of `dig`? It's pretty detailed (some may say, too much so) and if you read about DNS and read the output carefully, everything should fit nicely.

1. DNS is an important L7 protocol (by the OSI model). Let's start with it and spiral down.

   As fun as  `dig`  is, let's try to understand how the protocol itself looks like.
   To do that, we're going to [sniff](https://en.wikipedia.org/wiki/Packet_analyzer) the traffic between our DNS client and DNS server. BTW, this is good place to notice that when talking about networking, we're going to see the [client-server model](https://en.wikipedia.org/wiki/Client%E2%80%93server_model) **a lot**. Let's try to 

   ```sh
   sudo tcpdump -w /tmp/dns_dump.pcap -i lo port domain &
   dig i.am.not a @127.0.0.1 > /dev/null 2>&1
   pkill tcpdump
   tcpdump -nn -r /tmp/dns_dump.pcap 
   ```

   You probably get an output like this

   ```plain
   15:45:20.894040 IP 127.0.0.1.54618 > 127.0.0.1.53: 56221+ [1au] A? i.am.not. (49)
   15:45:20.894162 IP 127.0.0.1.53 > 127.0.0.1.54618: 56221*$ 1/0/1 A 127.0.0.3 (53)
   ```

   Because we're currently interested only in L7, let's filter everything from the other layers, and focus only on 

   ```plain
   A? i.am.not.
   A 127.0.0.3
   ```

   * Why `port domain`? See `/etc/services` and `man 5 services`.
      * Why `man 5`? Read `man man`
   * Can we correlate the output of `tcpdump` to the output of `dig`?

1. Now we think we understand how a typical L7 service looks like. Let's try to dive deeper and undertand how come it manges to do it. Let's look at [L4, the transport layer](https://en.wikipedia.org/wiki/Transport_layer)

   As you might have noticed, the DNS query and response were performed over port 53 using [UDP](https://en.wikipedia.org/wiki/User_Datagram_Protocol). We're going to find that this protocol has it's limitations.

   Let's push a new configurtion to `dnsmasq`. I want `i.am.not` to resolve to every address of the pattern `127.0.[0-2].[3-254]`. To do so, we'll push this configuration

   ```sh
   sudo cat > /etc/dnsmasq.conf <<EOF
   user=dnsmasq
   group=dnsmasq
   no-hosts
   addn-hosts=/tmp/i.am.not
   EOF
   ```

   Now, the file `/tmp/i.am.not` should include every address with the pattern requested, in this format

   ```sh
   [user@localhost ~]$ head -2 /tmp/i.am.not 
   127.0.0.3 i.am.not
   127.0.0.4 i.am.not
   ```

   Try again the trick with `tcpdump` and `dig`. Can you see where it stopped being UDP and started being TCP? Why did that happen?

   You may also want to explore the TCP flow and see if you understand the messages. You can [use this blog post](https://bencane.com/2014/10/13/quick-and-practical-reference-for-tcpdump/) for help.

1. We can clearly see that `dnsmasq` serves both UDP and TCP, by running

   ```plain
   [user@localhost ~]$ netstat -lnp | grep 53
   tcp        0      0 0.0.0.0:53              0.0.0.0:*               LISTEN      16001/dnsmasq       
   tcp6       0      0 :::53                   :::*                    LISTEN      16001/dnsmasq       
   udp        0      0 0.0.0.0:53              0.0.0.0:*                           16001/dnsmasq       
   udp6       0      0 :::53                   :::*                                16001/dnsmasq       
   ```

   However, this innocent looking output tells us another thing; `dnsmasq` is totally exposing us to attacks from the internet! 
   * `man netstat`, BTW
   * Why is binding to `0.0.0.0` such a bad idea?
   * Notice that there's also IPv6 here. We'll disable it later.

   To understand if we're really open to receive requests from the internet, let us consider the Linux firewall - `iptables`

   Run ```iptables -L INPUT``` to understand what kind of traffic your workstation is willing to receive. Output may vary, depending on your setup. As always `man iptables` is your friend here.
   * Are you in a crisis? Are you really open to receive DNS queries from the internet?

1. We can bind `dnsmasq` to a specific IP address, `127.0.0.1`, for instance, thus eliminating the possibility of someone querying us from the internet, even without the protection of `iptables`. However, this isn't nearly fun enough, and won't teach us how different client and server talk.

   Let's create, instead, a new IP address and bind `dnsmasq` to it. We'll do it in a [subnet](https://en.wikipedia.org/wiki/Subnetwork) totally different from the one we live in.
   * Will  this address be accessible from the internet? Can you explain your answer?

   So our first stop is to understand what is our "primary" device. Many a time it'll be `eth0`, but let's be sure.

   ```sh
   NIC=$(ip route show default | sed 's/.*dev \([^ ]*\) .*$/\1/') ; echo $NIC
   ```

   * Can you explain this `sed` expression?

   Now it's time to create a new address on this device

   ```sh
   ip address add 172.172.172.1/24 dev ${NIC}
   ```

   and bind ndsmasq to it

   ```sh
   echo listen-address=172.172.172.1 >> /etc/dnsmasq.conf
   service dnsmasq restart
   dig i.am.not a @127.0.0.1
   dig i.am.not a @172.172.172.1
   ```

1. While it's true that we've bound `dnsmasq` to a fictious IP, we still don't have an understanding of how communication works "in the wild".

   Let's try to give `dnsmasq` it's very own network interface card.

   ```sh
   ip link add nicK type dummy
   ip addr del 172.172.172.1 dev $NIC
   ip addr add 172.172.172.1/24 dev nicK
   dig i.am.not a @172.172.172.1
   ```

   Well, that's nice, but does it actually take us farther in our journey to understand network communications? Probably not very far. We have created an L2 device, but it isn't connected to anything.

   Actually, when we think about it, the first image that comes to mind when we think about networks is lots and lots of cables. So let's create a cable! What is a cable if not two ends and something uninteresting in the middle?

   ```sh
   ip addr del 172.172.172.1 dev nicK
   ip link del dev nicK
   ip link add nicK0 type veth peer name nicK1
   ip link set nicK0 up
   ip link set nicK1 up
   ip link add 172.172.172.1 dev nicK0
   ```

   So now we have two sides of a cable. One of them has an ip address, which `dnsmasq` is bound to. What's happening to the other side of the cable, to nicK1? It's just flapping around. Let's give it an IP address. We'll then try to force `dig` to use this address to query `dnsmasq`, and so we'll have two components talking to each other over a cable. Thats sounds more like how network communication works.

   ```sh
   ip addr add 172.172.172.2/24 dev nicK1
   dig -b 172.172.172.2 i.am.not a @172.172.172.1
   ```

   While we sit smugly and see that everything still works as expected, let doubt crawl into our hearts. Is it possible that we don't really simulate a network just yet? Try to use `tcpdump` to see if something is actually going on on this cable.

1. Our smart computer knows that 172.172.172.1 is one of its addresses, so the bustard never bother to use the shiny cable we gave it, and uses `lo`, the [loopback device](https://en.wikipedia.org/wiki/Loopback). But even we it didn't, let's try to picture what we've got so far - we have our computer, and we have a cable, and both sides of the cable are connected to the same computer. Let's try to change this picture.

   Network namespaces are Linux way to pretend it's more than one computer. Only it beleives its lie, but sometimes it's enough. It's in the heart of both Linux containers, virtualization and cloud. If we could put `dig` and `dnsmasq` in different namespaces, we'll have a new network topology; two computers connected "back to back"

   ```sh
   ip netns add ns1
   ip link set nicK1 netns ns1 up
   ip --netns ns1 address add 172.172.172.2/24 dev nicK1
   dig -b 172.172.172.2 i.am.not a @172.172.172.1
   ip netns exec ns1 dig -b 172.172.172.2 i.am.not a @172.172.172.1
   ```

   * Can you explain why our first attempt at `dig` didn't work?
   * Try to prove with `tcpdump` that we do use the cable now.
   * Can you run `dnsmasq` in it's own network namespace too?

1. You should be proud! You have a mini-network running locally on your computer.

   However, as you may be aware, most computer networks aren't two computers connected back-to-back. It doesn't scale well. We use [switches](https://en.wikipedia.org/wiki/Network_switch) for that. Since we have a virtual cable, let's connect it to a virtual switch. Linux has some [fancy virtual switches](https://www.openvswitch.org/) available out there, but we'll use the built-in one.

   ```sh
   # clean up previous run
   ip address delete 172.172.172.1 dev nicK0
   ip netns delete ns1
   ip link delete nicK0 ; ip link delete nicK1
   service dnsmasq stop

   # create two veth paris and two namespaces
   ip netns add nsclient
   ip netns add nsserver
   ip link add clientR type veth peer name clientL
   ip link add serverR type veth peer name serverL

   # create a bridge and connect the local ends to it
   ip link add br0 type bridge
   ip link set br0 up
   ip link set clientL master br0 up
   ip link set veth-nsservert-local master br0 up

   # send remote ends to their namespaces
   ip link set clientR netns nsclient
   ip link set serverR netns nsclient
   ip --netns nsclient address add 172.172.172.2/24 dev clientR
   ip --netns nsserver address add 172.172.172.1/24 dev serverR
   ip netns exec nsserver dnsmasq
   ip netns exec nsclient dig i.am.not a @172.172.172.1
   ```

1. It's quite an amazing thing we've done here. We have two 'computers' connected by a switch.

   Now, when the client talks to the server, we know how the client knows the server's IP address - we told him what it is. However, there's another address we haven't talked about.

   * Try to sniff traffic between the client and the server using `tcpdump`. Use its flags `-e` and `-i`. Can you figure out how? What does `-e` means?
   * Try to run `ip --netns nsserver link show`. Does the output from `tcpdump -e` makes more sense?

   The output of `tcpdump` should have shown you ARP requests and responses. ARP is the protocol we use to translate IP addresses to MAC addresses. To understand more aobut MAC addresses, read about Ethernet.

   Learned addresses are cached. We can view them with `ip netns exec nsclient arp -a`.

   Another interesting fact to know, is that network switches also learn about MAC addresses. However, they have no interes in associating MAC addresses to IP addresses. Therefore, they don't have ARP tables, but CAM tables. You can see this table by running `brctl showmacs br0`

1. Why should we thing that the DNS server and client, in real world scenarios, are always nieghbors? It is often the case that the server and the client are segmented to different networks.

   We can simulate this by creating another virtual switch, and connecting the server and the client to different switches. Then, we'll connect a 3rd namespace to both switches. This namspace will serve as our router.

   First thing, tear down the previous setup. Then

   ```sh
   # create two veth paris and two namespaces
   ip netns add nsclient
   ip netns add nsserver
   ip netns add nsrouter
   ip link add clientR type veth peer name clientL
   ip link add serverR type veth peer name serverL
   ip link add routerR0 type veth peer name routerL0
   ip link add routerR1 type veth peer name routerL1

   # create a bridge and connect the local ends to it
   ip link add br0 type bridge
   ip link add br1 type bridge
   ip link set br0 up
   ip link set br1 up
   ip link set clientL master br0 up
   ip link set serverL master br1 up
   ip link set routerL0 master br0 up
   ip link set routerL1 master br1 up

   # send remote ends to their namespaces
   ip link set clientR netns nsclient up
   ip link set serverR netns nsserver up
   ip link set routerR0 netns nsrouter up
   ip link set routerR1 netns nsrouter up
   ip --netns nsclient address add 172.172.173.2/24 dev clientR
   ip --netns nsserver address add 172.172.172.2/24 dev serverR
   ip --netns nsrouter address add 172.172.173.1/24 dev routerR0
   ip --netns nsrouter address add 172.172.172.1/24 dev routerR1
   ```

   Now, if you'll try to query the DNS serveer from the client, what do you thing will happen? Try it out.

   What's missing is the knowlege of the client and the server how to get to one another. When the were connected to the same bridge, they could find each other. More formally, they were in the same [broadcast domain](https://en.wikipedia.org/wiki/Broadcast_domain).

   * How does the concept of broadcast domain relates to what you saw when you analyzed ARP?

   To tell the client and the server how to reach each other, we'll add an entry to their [routing tables](https://en.wikipedia.org/wiki/Routing_table)

   ```sh
   ip --netns nsclient route add 172.172.172.0/24 via 172.172.173.1
   ip --netns nsserver route add 172.172.173.0/24 via 172.172.172.1
   ```

   * If you still find that you can't query the server from the client, try running `sysctl -a | grep -e "net.ipv4..*\.forwarding" | cut -d' ' -f 1 | while read conf ; do sysctl $conf=1 ; done`.

## Advanced

1. Want to turn your router to a simple firewall? Run `man iptables`

   * Yes, `iptables`. `firewalld` is a lie.

1. Try using `openvswitch` instead of `bridge`. It let's you introduce the concept of [vLAN](https://en.wikipedia.org/wiki/Virtual_LAN), so instead of `br0` and `br1` you can use a single switch.

1. Let's disable IPv6! Who needs it, anyway? (Well, the world)

   ```sh
   sysctl -a | grep ipv6 | grep disable
   ```

   Can you take it from here?

1. Want to build a container from scratch? [Here's your chance](https://github.com/fewbytes/rubber-docker)