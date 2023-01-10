wg-autoconf: easy tool to setup wireguard in the typical client/server use case

Usage:
  ./wg-autoconf [OPTIONS] <COMMAND>

OPTIONS
  -i <interface>    Set name of the wireguard-interface.
                    Defaults to: wg-autoconf
  -v                Increase verbosity.

CONNECT OPTIONS
  -u <unique-id>    Set unique identifier as client. The server must first authorize clients by adding
                    their identifier to its authorization file (/etc/wireguard/<interface>.auth).
                    Defaults to the public key of the client.
  -k <seconds>      Set persistent keepalive seconds.
                    Defaults to: 25

LISTEN OPTIONS
  -p <ipv6-prefix>  Set prefix of the virtual IPv6 network.
                    Defaults to: 7767:0:0
  -s <ipv6-subnet>  Set subnet of the virtual IPv6 network.
                    Defaults to: :1
  -a <ipv6-address> Set address of the server in the virtual IPv6 network
                    Defaults to: ::1
  -b <bind-host>    Set host to bind to.
                    Defaults to: 0.0.0.0

COMMANDS

  connect <host> [port]

      Connect to the given server running wg-autoconf listen.
      Port defaults to 51820.

  listen [port]

      Listen at the given TCP-port.


