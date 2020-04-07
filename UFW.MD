### Solving UFW and Docker issues

This solution needs to modify only one UFW configuration file, all Docker configurations and options remain the default.

Modify the UFW configuration file `/etc/ufw/after.rules` and add the following rules at the end of the file:

    # BEGIN UFW AND DOCKER
    *filter
    :ufw-user-forward - [0:0]
    :DOCKER-USER - [0:0]
    -A DOCKER-USER -j RETURN -s 10.0.0.0/8
    -A DOCKER-USER -j RETURN -s 172.16.0.0/12
    -A DOCKER-USER -j RETURN -s 192.168.0.0/16

    -A DOCKER-USER -p udp -m udp --sport 53 --dport 1024:65535 -j RETURN

    -A DOCKER-USER -j ufw-user-forward

    -A DOCKER-USER -j DROP -p tcp -m tcp --tcp-flags FIN,SYN,RST,ACK SYN -d 192.168.0.0/16
    -A DOCKER-USER -j DROP -p tcp -m tcp --tcp-flags FIN,SYN,RST,ACK SYN -d 10.0.0.0/8
    -A DOCKER-USER -j DROP -p tcp -m tcp --tcp-flags FIN,SYN,RST,ACK SYN -d 172.16.0.0/12
    -A DOCKER-USER -j DROP -p udp -m udp --dport 0:32767 -d 192.168.0.0/16
    -A DOCKER-USER -j DROP -p udp -m udp --dport 0:32767 -d 10.0.0.0/8
    -A DOCKER-USER -j DROP -p udp -m udp --dport 0:32767 -d 172.16.0.0/12

    -A DOCKER-USER -j RETURN
    COMMIT
    # END UFW AND DOCKER

Using command `sudo systemctl restart ufw` or `sudo ufw reload` to restart UFW after changing the file. Now the public network can't access any published docker ports, the container and the private network can visit each other normally, and the containers can also access the external network from inside. **There may be some unknown reasons cause the UFW rules will not take effect after restart UFW, please reboot servers.**

If you want to allow public networks to access the services provided by the Docker container, for example, the service port of a container is `80`. Run the following command to allow the public networks to access this service:

    ufw route allow proto tcp from any to any port 80

This allows the public network to access all published ports whose container port is `80`.

Note: If we publish a port by using option `-p 8080:80`, we should use the container port `80`, not the host port `8080`.

If there are multiple containers with a service port of `80`, but we only want the external network to access a certain container. For example, if the private address of the container is `172.17.0.2`, use the following command:

    ufw route allow proto tcp from any to 172.17.0.2 port 80

If the network protocol of a service is UDP, for example a DNS service, you can use the following command to allow the external network to access all published DNS services:

    ufw route allow proto udp from any to any port 53

Similarly, if only for a specific container, such as IP address `172.17.0.2`:

    ufw route allow proto udp from any to 172.17.0.2 port 53

## How it works?

The following rules allow the private networks to be able to visit each other. Normally, private networks are more trusted than public networks.

    -A DOCKER-USER -j RETURN -s 10.0.0.0/8
    -A DOCKER-USER -j RETURN -s 172.16.0.0/12
    -A DOCKER-USER -j RETURN -s 192.168.0.0/16

The following rules allow UFW to manage whether the public networks are allowed to visit the services provided by the Docker container. So that we can manage all firewall rules in one place.

    -A DOCKER-USER -j ufw-user-forward

The following rules block connection requests initiated by all public networks, but allow internal networks to access external networks. For TCP protocol, it prevents from actively establishing a TCP connection from public networks. For UDP protocol, all accesses to ports which is less then 32767 are blocked. Why is this port? Since the UDP protocol is stateless, it is not possible to block the handshake signal that initiates the connection request as TCP does. For GNU/Linux we can find the local port range in the file `/proc/sys/net/ipv4/ip_local_port_range`. The default range is `32768 60999`. When accessing a UDP protocol service from a running container, the local port will be randomly selected one from the port range, and the server will return the data to this random port. Therefore, we can assume that the listening port of the UDP protocol inside all containers are less then `32768`. This is the reason that we don't want public networks to access the UDP ports that less then `32768`.

    -A DOCKER-USER -j DROP -p tcp -m tcp --tcp-flags FIN,SYN,RST,ACK SYN -d 192.168.0.0/16
    -A DOCKER-USER -j DROP -p tcp -m tcp --tcp-flags FIN,SYN,RST,ACK SYN -d 10.0.0.0/8
    -A DOCKER-USER -j DROP -p tcp -m tcp --tcp-flags FIN,SYN,RST,ACK SYN -d 172.16.0.0/12
    -A DOCKER-USER -j DROP -p udp -m udp --dport 0:32767 -d 192.168.0.0/16
    -A DOCKER-USER -j DROP -p udp -m udp --dport 0:32767 -d 10.0.0.0/8
    -A DOCKER-USER -j DROP -p udp -m udp --dport 0:32767 -d 172.16.0.0/12

    -A DOCKER-USER -j RETURN

If a docker container doesn't follow the OS's settings when receiving data, that is to say, the minimal port number less than `32768`. For example, we have a Dnsmasq container. The minimal port number that Dnsmasq uses for receiving data is `1024`. We can use the following command to allow a bigger port range used for receiving DNS packages.

    ufw route allow proto udp from any port 53 to any port 1024:65535

Because DNS is a very common service, so there is already a firewall rule to allow a bigger port range to receive DNS packages.

ufw allow from 172.16.0.0/12 to any
