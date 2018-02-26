# killermoehre.github.io

Some thoughts I had …

# 2018-02-26: How to make the haproxy stats socket network available (with systemd)

## Problem

With my current customer we got the problem that our apache webservers behind a haproxy stalled during logrotate. Therefore timeouts are reached and API calls misses.

Sure, one could set

    copytruncate

in the rsyslog configuration, but there is the slight chance of missing messages. And with high output logs (we log _every_ request) this risk can't be taken.

So my approach is to tell haproxy that the backend server is going to logrotate and just take it out of the loadbalancing. Logrotate provides with `prerotate/endscript` and `postrotate/endscript` the necessary tools to implement this on the backend side. The problem here is that haproxy has only a local socket as API, not a IP:PORT. The option for this is

    global
      stats socket /run/haproxy.stat

## Solution

So how can I talk to haproxy from some random backend host via TCP/IP? Sure, one could setup some SSH keys, but this would be combersome, as now you have to exchange keys. Instead, the solution I propose is to make the local haproxy stats socket available on a local port. For this we use `systemd-socket-proxyd`. I will use `/run/haproxy.sock` as socket, `10.0.0.3:20000` as network address and `foo-api` as haproxy backend name as examples in this document. See [man:systemd.socket](man:systemd.socket) for more information.

`systemd-socket-proxyd` works in combination with a `.socket` unit.<sup>[1](#myfootnote1)</sup> The `.socket` activates a `.service` which forwards to the actual haproxy.

    === proxy-to-haproxy.socket ===
    [Socket]
    ListenStream=10.0.0.3:20000
    [Install]
    WantedBy=socket.target

    === proxy-to-haproxy.service ===
    [Unit]
    Requires=haproxy.service
    After=haproxy.service
    Requires=proxy-to-haproxy.socket
    After=proxy-to-haproxy.socket
    [Service]
    ExecStart=/usr/lib/systemd/systemd-socket-proxyd /run/haproxy.sock
    PrivateTmp=true
    PrivateNetwork=true

To enable this configuration, put those files into `/etc/systemd/system/`, run `systemctl daemon-reload` and `systemctl enable --now proxy-to-haprxy.socket`. The `/run/haproxy.sock` is now available from the network and can be reached via `telnet`. To check this, one can use

    echo 'show stats' | telnet 10.0.0.3 20000

This gives you the current statistics of the haproxy. To combine this with logrotate the appropriate commands have to be put in the actual `logrotate.conf` in the `prerotate` / `postrotate` sections.

    prerotate
        echo "disable server foo-api/$HOSTNAME" | telnet 10.0.0.3 20000
        sleep 5
    endscript
    postrotate
        echo "enable server foo-api/$HOSTNAME" | telnet 10.0.0.3 20000
    endscript

The `sleep 5` is just as a security measurement to make sure that the backend server can finish serving it's clients.

## Caveats

The exposed haproxy socket can be accessed by everybody and everything able to reach the ip/port. There is no authorization, authentication, nor encryption, so make sure your firewall is tight. Also if all your backend hosts rotate at the exact same time, you will basically shut down your service.

## Footnotes

<a name="myfootnote1">1</a>: You could also use it without `.socket` by using the special syntax `ExecStart=@/usr/lib/systemd/systemd-socket-proxy $IP:$PORT /run/haproxy.stat`
