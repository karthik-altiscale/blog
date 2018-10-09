
### Emulating TCP States on linux

### TCP States diagram

There are several articles, drawings and explanations in the internet about the various TCP states. [Here](https://upload.wikimedia.org/wikipedia/en/5/57/Tcp_state_diagram.png) is one from wikipedia

There are so many reasons a TCP connection can go into those states. I am attempting make the TCP connection to go to each of those states using IPTables and other hacks and observe using tcpdump and netstat.

### Setup  
Pretty straight forward setup. I have 2 VMs in a LAN
* Client with ip 192.168.2.10, which runs `telnet server_ip 80`
* Server with ip 192.168.2.11, which listens on port 80 by running `nc -l 80`

&nbsp;

### State: LISTEN
----
**Who goes to this state:** Server

There are countless ways to start LISTENing on `IP:Port`. I am using `nc -l 80`
  
**Result**

![Alt text]({{ site.baseurl }}/assets/img/listen.png)

&nbsp;

### State: SYN SENT
----

**Who goes to this state:** Client

**When:** a client initiates a 3 way handshake but there is no response 

> well there may be [[R] (reset) response](https://tools.ietf.org/html/rfc793#page-36) or ICMP if server uses `-j REJECT`

**Simulate:** We can simulate this by dropping the SYN (actually all) packets on port 80 on the server

  
```[root@B ~]# iptables -A INPUT  -p tcp --destination-port 80 -j DROP```

and start client

```
[root@Client ~]# telnet 192.168.2.11 80
Trying 192.168.2.11...
```

**Observe:** tcpdump on *Client* shows syn [S] packet is sent and since no response there are resends

![Alt text]({{ site.baseurl }}/assets/img/tcpdump_syn_sent_client.png)

oh btw B's tcpdump's pcap library still can see incoming [S] (the sk_buff headers) but filtered and dropped by our IPTables rule and so the packets are never delivered to the protocol layer ([detailed explanation here](https://blog.packagecloud.io/eng/2016/06/22/monitoring-tuning-linux-networking-stack-receiving-data/#__netif_receive_skb_core-delivers-data-to-packet-taps-and-protocol-layers)).

![Alt text]({{ site.baseurl }}/assets/img/tcpdump_syn_sent_server.png)

If you don't want Server to see anything you can block outgoing packets to port 80 in Client itself

```[root@Client ~]# iptables -A OUTPUT -p tcp --destination-port 80 -j DROP```

**Result:**
Client has sent [S] (syn) and is now in *SYN_SENT* state

![Alt text]({{ site.baseurl }}/assets/img/netstat_syn_sent_client.png)

&nbsp;

### State: SYN RECEIVED
----
**Who goes to this state:** Server

**When:** Server receives [S] (syn) and is interested in responding back with [S.] (syn/ack) but client is dead to receive the [S.]

> wait a second !!! why does it sound familiar ?????? That's **SYN FLOOD**

**Simulate:** We can simulate this situation by dropping the packets coming from the server (please make sure to `iptables -F`)

```[root@Client ~]# iptables -A INPUT -p tcp --source-port 80 -j DROP```

and start client

```
[root@Client ~]# telnet 192.168.2.11 80
Trying 192.168.2.11...
```

**Observe:**
* tcpdump on Client shows [S] (syn) packet is sent and Server responds with [S.] (syn/ack)

![Alt text]({{ site.baseurl }}/assets/img/tcpdump_syn_recv_client.png)

* Client side pcap sees [S.] but dropped by the IPTables rule. Packets never reach the protocol layer ([detailed here](https://blog.packagecloud.io/eng/2016/06/22/monitoring-tuning-linux-networking-stack-receiving-data/#__netif_receive_skb_core-delivers-data-to-packet-taps-and-protocol-layers)).
  * alternatively you may block [S.] on server itself `[root@Server] iptables -A OUTPUT -p tcp --source-port 80 -j DROP`

**Result:**

Server has sent [S.] (syn/ack) but no [.] (ack) from client is received yet. Server goes into *SYN_RECV* state
  
![Alt text]({{ site.baseurl }}/assets/img/netstat_syn_recv_server.png)

&nbsp;

### State: ESTABLISHED
----

**Who goes to this state:** Both server and client

**When:** 3 way handshake is successfully completed, both sides' TCP connection state becomes ESTABLISHED

**Simulate:** On both server and client `iptables -F` (or `-j ACCEPT` on source/dest port 80)

**Observe:**
Server side 3 way handshake went successful

![Alt text]({{ site.baseurl }}/assets/img/tcpdump_established_server.png)

Client side 3 way handshake went successful

![Alt text]({{ site.baseurl }}/assets/img/tcpdump_established_client.png)

**Result:**

*ESTABLISHED* on Server

![Alt text]({{ site.baseurl }}/assets/img/netstat_established_server.png)

*ESTABLISHED* on Client

![Alt text]({{ site.baseurl }}/assets/img/netstat_established_client.png)

&nbsp;

### State: FIN WAIT 1
----

**Who goes to this state:** whoever initiates connection termination (in our case Client)

**When:**  After 3 way handshake (and data transfer), 
* client wants to terminate the TCP connection sends a [F] (fin) packet (*i am not very sure but I think its bundled with an ack of previous transaction and hence we see a [F.]*) 
* client goes into FIN_WAIT1 state. 

If server does not respond with an [.] (ack) (*generally its [F.] instead of [.] i.e bundled with server's fin detailed below*) client continues to stay in FIN_WAIT1 state

  **Simulate:**

Remove all rules `[root@Server ~]# iptables -F`  so that the 3 way handshake happens successfully

```
[root@Client ~]# telnet 192.168.2.11 80
Trying 192.168.2.11...
Connected to 192.168.2.11.
Escape character is '^]'.
```
now block the port 80 on server

```
[root@Server ~]# iptables -A INPUT -p tcp --destination-port 80 -j DROP
```

and kill the client

```
^]q

telnet> q
Connection closed.
```

**Observe:**

Client sends [F] (fin) packet

![Alt text]({{ site.baseurl }}/assets/img/tcpdump_fin_wait_client.png)

but since no [.] (ack) response from server, client keeps resending [F]

**Result:**
The *Client* goes into FIN_WAIT state till it hears back an ack from server

![Alt text]({{ site.baseurl }}/assets/img/netstat_fin_wait_client.png)

*Note:* the server still thinks socket is open and it can send or receive packets and we have an half open tcp connection. This is bad

![Alt text]({{ site.baseurl }}/assets/img/netstat_fin_wait_server.png)

&nbsp;

### State: CLOSE WAIT
----

**Who goes to this state:** whoever receives the first [F] (termination request). In our case Server

**When:**  After 3 way handshake (and data transfer), 
1. client wants to terminate the TCP connection sends a [F] (fin) packet and goes into FIN_WAIT1 state
2. server receives F (fin) and goes to CLOSE_WAIT state and sends back an [.] (ack)
3. server wants to terminate the TCP connection sends a [F] (fin) packet by running `close()`
    * If for some reason server (who has received client side [F]) can not run `close()` it continues to be in CLOSE_WAIT

**Simulate:**

To simulate this we need a buggy code that after receiving FIN packet WON'T run close() on that TCP socket ... [here is a detailed blog](https://blog.cloudflare.com/this-is-strictly-a-violation-of-the-tcp-specification).

Actually I found another way to reproduce *socket leakage*

As usual on the server `nc -l 80`
and 
on the client `telnet 192.168.2.11 80`
you see ESTABLISHED connection with PID

![Alt text]({{ site.baseurl }}/assets/img/netstat_close_wait_established.png)

now open one more connection from client `telnet 192.168.2.11 80`. There are 2 ESTABLISHED connections 

![Alt text]({{ site.baseurl }}/assets/img/netstat_close_wait_established_2.png)
if you notice the second connection *with port 40658* is NOT bound to any process. I am not sure if this is a bug in `nc` or this is how its supposed to work.

I tried `strace` and `strace -f -p shell_pid`, no code executes (and hence no `close()` called) when I kill the client side tcp socket with port 40658 

When I do so, tcpdump shows server receives [F] (fin) packet and responses with [.] (ack), Server **does not bundle its [F] (fin**)

![Alt text]({{ site.baseurl }}/assets/img/tcpdump_close_wait_server.png)

**Result:**
and hence waiting for close()

![Alt text]({{ site.baseurl }}/assets/img/netstat_close_wait_server.png)

&nbsp;

### State: FIN WAIT 2
----

**Who goes to this state:** party who was in FIN WAIT 1  

**When:**  After 3 way handshake (and data transfer), 
1. client wants to terminate the TCP connection sends a [F] (fin) packet and goes into FIN_WAIT1 state
2. server receives F (fin) and goes to CLOSE_WAIT state and sends back an [.] (ack)
3. server wants to terminate the TCP connection from server-side sends a [F] (fin) packet by running `close()`
    * If for some reason server (who has received client side [F]) can not run `close()` it continues to be in CLOSE_WAIT
    * In that case server sends only [.] (ack)
4. client receives the [.] (ack) and goes from FIN_WAIT1 to FIN_WAIT2

**Result**

Waiting for [F] (fin) from server side ie. waiting for server side termination process

![Alt text]({{ site.baseurl }}/assets/img/netstat_close_wait_client_fin_wait2.png)

&nbsp;

### State: LAST ACK
----
**Who goes to this state:** party who was in CLOSE_WAIT (in our case Server)

**When:**  After 3 way handshake (and data transfer), 
1. client wants to terminate the TCP connection sends a [F] (fin) packet and goes into FIN_WAIT1 state
2. server receives [F] (fin) and goes to CLOSE_WAIT state and sends back an [.] (ack)
3. server wants to terminate the TCP connection from server-side sends a [F] (fin) packet by running `close()`
4. server actually bundles 2 and 3 and sends [F.] (fin/ack) and goes from CLOSE_WAIT to LAST_ACK state
    * tcpdump shows up as 3 way termination, in theory tcp termination is a 4 way process

server stays in LAST_ACK state till it receives a final ack for the server side [F]

**Simulate:** Remove all rules `[root@Server ~]# iptables -F`  so that the 3 way handshake happens successfully

```
[root@Client ~]# telnet 192.168.2.11 80
Trying 192.168.2.11...
Connected to 192.168.2.11.
Escape character is '^]'.
^]
```

block the port 
```[root@Client ~]# iptables -A INPUT -p tcp --source-port 80 -j DROP```

**Observe:**
[F.] being sent from server to client but client does not ack 

![Alt text]({{ site.baseurl }}/assets/img/tcpdump_last_ack_server.png)

**Result:**

![Alt text]({{ site.baseurl }}/assets/img/netstat_last_ack_server.png)

&nbsp;

### State: TIME WAIT
----

**Who goes to this state:** party who initated the tcp termination first

**When:** the tcp termination is successfully done on both sides

**Simulate:** Remove all rules `[root@Server ~]# iptables -F`  so that the 3 way handshake happens successfully

```
[root@Client ~]# telnet 192.168.2.11 80
Trying 192.168.2.11...
Connected to 192.168.2.11.
Escape character is '^]'.
^]q

telnet> q
Connection closed.

```

**Result**

![Alt text]({{ site.baseurl }}/assets/img/netstat_time_wait.png)


-----

I am using issues as [Comments Section](https://github.com/karthik-altiscale/blog/issues/1) 
