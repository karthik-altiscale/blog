
## TCP States diagram

  There are several articles, drawings and explanations in the internet about the various TCP states, and there are many reasons TCP can go into those states.

  I am attempting to use IPTables rules that block/allow packets making the TCP connection to go to particular state and view it using `netstat`

## Setup  
  Pretty straight forward setup. I have 2 VMs in a LAN
  * Client which runs `telnet server_ip 80`
  * Server which listens on port 80 by running `nc -l 80`

## State: LISTEN
----

  Nothing much to explain here, server is LISTENing on port any_ip:80
  
  ![Alt text]({{ site.baseurl }}/assets/img/listen.png)


## State: SYN SENT
----
  This happens when a client initiates a 3 way handshake but there is not response (btw no response is different from a reset [R] response or a Rejected icmp)

  We can simulate this by dropping the SYN (actually all) packets on port 80 on the server
  
  ```[root@B ~]# iptables -A INPUT  -p tcp --destination-port 80 -j DROP```

  Running ```telnet server 80```, tcpdump on **Client** shows syn [S] packet is sent and since no response there are resends

  ![Alt text]({{ site.baseurl }}/assets/img/tcpdump_syn_sent_client.png)

  oh btw B's tcpdump's pcap library still can see incoming [S] (the sk_buff headers) but filtered and dropped by our IPTables rule and so the packets are never delivered to the protocol layer ([detailed explanation here](https://blog.packagecloud.io/eng/2016/06/22/monitoring-tuning-linux-networking-stack-receiving-data/#__netif_receive_skb_core-delivers-data-to-packet-taps-and-protocol-layers)).

  ![Alt text]({{ site.baseurl }}/assets/img/tcpdump_syn_sent_server.png)

  Client has sent [S] and now is in **SYN_SENT** state

  ![Alt text]({{ site.baseurl }}/assets/img/netstat_syn_sent_client.png)

