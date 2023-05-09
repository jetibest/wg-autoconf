# Introduction

This tool automatically deals with the virtual IPv6 addresses.
Including using `ip -6 route` to setup routes, and `ip6tables` to setup (forwarding) rules for the firewall.
Furthermore, this tool can update the extended configuration file through cli-commands.

# Installation

    cd /tmp
    git clone https://github.com/jetibest/wg-autoconf
    cp wg-autoconf/wg-autoconf /usr/local/bin/wg-autoconf

# Example usage

    # exchange public keys
    s1> wg-autoconf server allow "<s2 pubkey>" Type=server
    s1> wg-autoconf server allow "<c1 pubkey>"
    s2> wg-autoconf server allow "<s1 pubkey>" Type=server
    s2> wg-autoconf server allow "<c1 pubkey>"
    
    # setup server TCP-daemons and interconnect the mesh network
    s1> wg-autoconf server listen
    s2> wg-autoconf server connect s1.example.org
    s2> wg-autoconf server listen
    
    # let a client connect to either server
    c1> wg-autoconf client connect mesh.example.org

where `s1` and `s2` are servers. `mesh.example.org` resolves to both `s1` or
`s2` (round robin). And `c1` is a client that will connect to one of the
servers.

# Manual

```
wg-autoconf: Automated Wireguard configuration cli-tool.

DEPENDENCIES
 -> wg
 -> socat
 -> ip6tables
 -> flock, ip, grep, sed, cat (busybox)
 -> any modern Linux shell (bash)


USAGE
  ./wg-autoconf [OPTIONS] <server|client|conf|wg> <COMMAND>


OPTIONS
  -i <interface>       Set name of the wireguard-interface.
                       Defaults to: wg0
  -v                   Increase verbosity.
  -h, --help <context> Show this help. Context can be any of server, client,
                       or conf.
                       Defaults to: Show help for all contexts.


WG COMMANDS

  update

      Synchronize configuration file with the Wireguard interface, and ensure
      all ip6tables rules and ip routes are setup correctly. Does not bring
      the Wireguard interface up if it is down.

  up

      Same as update, but also ensures the Wireguard interface is brought up.

  down

      Bring the Wireguard interface down. All ip6tables and ip routes rules
      are also cleaned up.


SERVER COMMANDS

  connect <host> [port]

      Connect to the given server running wg-autoconf listen.
      To be run by a server. Will register itself (as a server) at the remote
      server. If the local public key is authorized by the remote server, the
      remote server will append a Peer-section with this local server as its
      endpoint.
      Port defaults to 51820.

  listen [port]

      Listen at the given TCP-port.
      Any remote clients and servers can connect, and if authorized, be added
      to the subnet that this server provides.
      Port defaults to 51820.

  allow <public-key> [Type=<server|client>]

      Allow Peer identified by the given Public Key to connect.
      Ensures a Peer-section in the configuration file.
      If Type is set to server, the Peer will be able to register its own
      Endpoint and AllowedIPs entries upon connect.

  deny <public-key>

      Deny Peer identified by the given Public Key from connecting.
      Removes a Peer-section from the configuration file.


SERVER CONNECT OPTIONS

  -k <seconds>      Set persistent keepalive seconds.
                    Defaults to: 25


SERVER LISTEN OPTIONS

  -a <ipv6-address> Set address of the server in the virtual IPv6 network
                    Defaults to: 7767:1::1/32
  -b <bind-host>    Set host to bind to.
                    Defaults to: ::
                    Use 0.0.0.0 to bind to every IPv4 interface.
                    Use :: to bind to every IPv6 interface.
                    In order to listen at both IPv4 and IPv6 interfaces, two
                    separate instances must be run.
  -s <mask>         Specify a bitmask for the subnet division. Any client that
                    is connected to the virtual network can access any other
                    client, but only if it is in the same subnet.
                    Defaults to: 48


CLIENT COMMANDS

  connect <host> [port]

      Connect to the given server running wg-autoconf listen.
      To be run by a client, any existing configuration for this interface
      will be completely overwritten. Do NOT run as server, since all
      existing clients will be disconnected, and manual configuration changes
      will be deleted.
      Port defaults to 51820.


CLIENT CONNECT OPTIONS

  -m [MAC-address]  Set the Interface ID part of the IPv6 address. If the
                    MAC-address is not defined, then it will default to the
                    first available network device, which is then cached for
                    subsequent connects in: /etc/wireguard/<interface>.id
                    The server is not guaranteed to incorporate the
                    Interface ID into the client's allowed IP address (i.e.
                    in case the ID is not unique).
  -k <seconds>      Set persistent keepalive seconds.
                    Defaults to: 25


CONF COMMANDS

  show

      Print the contents of the configuration file.

  get-interface [entry]

      Print the Interface-section of the configuration file.
      If an entry is specified, only the value of that entry is printed.
      If an entry has multiple values separated by a comma, these values
      are printed on separate lines.

  get-peer <public-key> [entry]

      Print the Peer-section of the configuration file, that contains a
      PublicKey entry with the given public-key value.
      If an entry is specified, only the value of that entry is printed.
      If an entry has multiple values separated by a comma, these values
      are printed on separate lines.

  strip

  list-peers

  set-peer


AUTOCONF CONFIGURATION FILE

wg-autoconf uses non-standard properties, which are by prefixed by a single
comment character (#). These actually have effect, and are not considered
comments for wg-autoconf, but are comments with regard to Wireguard. This
allows for complete compatibility with Wireguard.


AUTOCONF PROTOCOL

wg-autoconf listens on the same port as WireGuard. Since wg-autoconf uses
TCP, and WireGuard uses UDP, there is no conflict.

Within the virtual network, only IPv6-addresses are supported due to its
additional flexibility. For the real non-virtual network, both IPv4 and IPv6
are supported.


IPV6 FORWARDING AND IP6TABLES

Note that when using listen, IPv6 forwarding is enabled (and accept_ra is set
to 2). This is why, for security, the default policy of the FORWARD chain will
be set to DROP. That is, no IPv6 forwarding is allowed unless explicitly
accepted by a rule in the FORWARD chain of ip6tables.

Any ip6tables rules that are added by wg-autoconf, will be in a separate chain
and will be automatically cleaned up when running the down command.


RECOMMENDED IPV6 ALLOCATION

  <prefix> : <server> : <subnet> : <unused> : <interface-identifier>

Where:
 -> prefix is a unique identifier for this virtual private network.
 -> server is a unique identifier for each individual server (i.e. site).
 -> subnet isolates clients from each other that are in different subnets.
 -> unused is a further custom distinction of client groups, may be left zero.
 -> interface-identifier is a 64-bit identifier reserved to uniquely identify
    clients across multiple subnets and groups. By default this is the
    MAC-address of the client.


TROUBLESHOOTING

 1. Check if interface is up and has the intended virtual IPv6-address
 2. Check if IPv6-forwarding is enabled, using:

      cat /proc/sys/net/ipv6/conf/<wg0|all>/forwarding

 3. Check if the virtual IPv6-address is routed correctly, using:

      ip -6 route [dev wg0]

 4. Check if the route is blocked by netfilter with ip6tables:

      ip6tables -L

 5. Check from the client's perspective if the Endpoint IP-address in
    the [Peer] section that belongs to the server is correct.
 6. Check if the port in the Endpoint from step 5 is correct.
 7. Check if the port from step 6 matches the ListenPort on the server:
    wg show <wg0> listen-port
 8. Check if the PublicKey entry of the [Peer] section matches the
    PublicKey generated from the PrivateKey of the [Interface] section
    of the other peer. Check both ends. If one end is misconfigured,
    then the whole connection fails.

Wireguard uses the UDP-protocol, the wg-autoconf utility uses TCP-protocol.
Debug UDP traffic using:

    tcpdump -n udp

If ping to a virtual address fails with:

    Destination unreachable: Address unreachable

then the traffic is correctly locally routed through the Wireguard link
device, but Wireguard responds that the virtual target IPv6-address is
unreachable. Check the AllowedIPs of the relevant [Peer] section.

Wireguard will give a warning:

    warning: AllowedIP has nonzero host part

which can be safely ignored. It indicates a subnet mask catches more than
one specific IPv6-address, and thus for clarity it should end with zeroes
(::). But wg-autoconf specifies the address anyway, to store the IP-address
of the server itself.

If a server is added to the mesh, all previously connected peers (clients
and servers) must reconnect using wg-autoconf. So that the new IP-addresses
are propagated. If the mesh is dynamic, it is highly recommended to use
wg-monitor which automatically propagates updates through the mesh network
and furthermore keeps connections alive (to achieve redundancy).


```
