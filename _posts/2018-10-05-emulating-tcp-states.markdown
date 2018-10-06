
## TCP States diagram

  There are several articles, drawings and explanations in the internet about the various TCP states, and there are many reasons TCP can go into those states.

  I am attempting to use IPTables rules that block/allow packets making the TCP connection to go to particular state and view it using `netstat`

## Setup  
  Pretty straight forward setup. I have 2 VMs in a LAN
  * Client which runs `telnet server_ip 80`
  * Server which listens on port 80 by running `nc -l 80`


### State: LISTEN
----
  **who goes to this state:** Server

  Nothing much to explain here, server is LISTENing on port any_ip:80
  
  ![Alt text]({{ site.baseurl }}/assets/img/listen.png)


### State: SYN SENT
----

  **who goes to this state:** Client

  **When:** a client initiates a 3 way handshake but there is no response (btw no response is different from a reset [R] response or a Rejected ICMP)

  **Simulate:** We can simulate this by dropping the SYN (actually all) packets on port 80 on the server
  
  ```[root@B ~]# iptables -A INPUT  -p tcp --destination-port 80 -j DROP```

  **Observe:** Running ```telnet server 80```, tcpdump on *Client* shows syn [S] packet is sent and since no response there are resends

  ![Alt text]({{ site.baseurl }}/assets/img/tcpdump_syn_sent_client.png)

  oh btw B's tcpdump's pcap library still can see incoming [S] (the sk_buff headers) but filtered and dropped by our IPTables rule and so the packets are never delivered to the protocol layer ([detailed explanation here](https://blog.packagecloud.io/eng/2016/06/22/monitoring-tuning-linux-networking-stack-receiving-data/#__netif_receive_skb_core-delivers-data-to-packet-taps-and-protocol-layers)).

  ![Alt text]({{ site.baseurl }}/assets/img/tcpdump_syn_sent_server.png)

  **Result:**
  Client has sent [S] and now is in *SYN_SENT* state

  ![Alt text]({{ site.baseurl }}/assets/img/netstat_syn_sent_client.png)


### State: SYN RECEIVED
----
  **who goes to this state:** Server

  **When:** Server receives [S] and is interested in responding back with [S.] (syn/ack) but client is dead to receive the [S.] .... wait a second !!! why does it sound familiar ?????? Thats **SYN FLOOD**

  **Simulate:** We can simulate this situation by dropping the packets coming from the server 

  btw `iptables -F` for sanity
  

  ```[root@Client ~]# iptables -A INPUT -p tcp --source-port 80 -j DROP```

  **Observe:**

  Running ```telnet server 80```, tcpdump on *Client* shows syn [S] packet is sent and Server responds with [S.] (syn/ack) but Client side filtered and dropped by our IPTables rule and so the packets are never delivered to the protocol layer ([detailed explanation here](https://blog.packagecloud.io/eng/2016/06/22/monitoring-tuning-linux-networking-stack-receiving-data/#__netif_receive_skb_core-delivers-data-to-packet-taps-and-protocol-layers)).

  ![Alt text]({{ site.baseurl }}/assets/img/tcpdump_syn_recv_client.png)

  proof that server responds [S.] (syn/ack) and keeps resending since the ack is not received 

  ![Alt text]({{ site.baseurl }}/assets/img/tcpdump_syn_recv_server.png)

  **Result:**

  Server has sent [S.] but no [.] (ack) from client is received yet. Server goes into *SYN_RECV* state
  
  ![Alt text]({{ site.baseurl }}/assets/img/netstat_syn_recv_server.png)


### State: ESTABLISHED
----

  **who goes to this state:** Both server and client

  **When:** 3 way handshake is successfully completed, both sides' TCP connection state becomes ESTABLISHED

  btw please `iptables -F` (or `-j ACCEPT` on port 80)

  **Observe:**
  Server side 3 way handshake went successful

  ![Alt text]({{ site.baseurl }}/assets/img/tcpdump_established_server.png)

  Client side 3 way handshake went successful

  ![Alt text]({{ site.baseurl }}/assets/img/tcpdump_established_client.png)

  **Result:**

  **ESTABLISHED** on Server

  ![Alt text]({{ site.baseurl }}/assets/img/netstat_established_server.png)

  **ESTABLISHED** on Client

  ![Alt text]({{ site.baseurl }}/assets/img/netstat_established_client.png)

### State: FIN WAIT 1
----

  **who goes to this state:** whoever initiates connection termination (can be either server or client)

  **When:** After 3 way handshake (and data transfer), when client or server wants to terminate the TCP connection Client (in our example) sends a FIN and goes to FIN_WAIT1 state. If server does not respond with an ACK  client continues to  stay in FIN_WAIT1 state

  **Simulate:**
  
    Remove all rules 

``` [root@Server ~]# iptables -F ```

    so that the 3 way handshake happens successfully

```[root@Client ~]# telnet 192.168.2.11 80
Trying 192.168.2.11...
Connected to 192.168.2.11.
Escape character is '^]'.```

  now block the port 80 (or kill the server or )

```[root@Server ~]# iptables -A INPUT -p tcp --destination-port 80 -j DROP```

  **Observe:**

  Client sends [F] (fin) packet
  
  ![Alt text]({{ site.baseurl }}/assets/img/tcpdump_fin_wait_client.png)

  but since no [.] (ack) response from server client keeps resending [F]

  **Result:**
  The *Client* goes into FIN_WAIT state till it hears back an ack from server
  ![Alt text]({{ site.baseurl }}/assets/img/netstat_fin_wait_client.png)
  
  *Note:* the server still thinks the TCP socket is open and it can send or receive packets. This is bad
  ![Alt text]({{ site.baseurl }}/assets/img/netstat_fin_wait_server.png)
