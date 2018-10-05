
# Emulating TCP States

## TCP States diagram

  There are several articles, drawings and explanations in the internet about the various TCP states, and there are many reasons TCP can go into those states.

  I am attempting to use IPTables rules that block/allow packets making the TCP connection to go to particular state and view it using `netstat`

## Setup  
  Pretty straight forward setup. I have 2 VMs in a LAN
  * Client which runs `telnet server_ip 80`
  * Server which listens on port 80 by running `nc -l 80`

## State: LISTEN
  ![Alt text]({{ site.baseurl }}/assets/img/listen.png){:width="50%"}



{% assign image_files = site.static_files | where: "image", true %}
{% for myimage in image_files %}
  {{ myimage.path }}
{% endfor %}
