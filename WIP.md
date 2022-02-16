Notes
- Check out https://unix.stackexchange.com/a/443919. This is basically saying: use `strace` to work
    out what syscalls you need to make to emulate the various `ip` commands.
- Check this out:
  ```sh
  sudo ip netns exec vnet0 su - msk
  ```
  Opens a shell as a specific user. Should be able to do the `strace` thing to figure out how this
  works, too.
- Can specify interface settings; unsure what will work with virtual interfaces.
    https://www.shellhacks.com/setup-dns-resolution-resolvconf-example/#comment-17208
- Basically describes how to achieve what we want to achieve with our network namespace:
    https://serverfault.com/a/925349
- See the following for non-root namespacing:
  - https://serverfault.com/a/990381
  - https://github.com/containers/bubblewrap
  - could use `strace` to figure out what it's doing
- Check out Linux "user namespaces"
- HTTP rewriting? If the k8s pods are terminating HTTPS (instead of an ingress) then we'd like to
    support HTTP rewriting so that the host name on the certificate matches the host name supplied
    in the HTTP request. This is a later roadmap item.

Implementation
- Get a list of all services in the supplied/default namespace.
- Decide on an IP range (I dunno.. probably just a /8? Do some research on this.). Expose this
    IP as a configuration option to the user. For example, we might decide to use 10.0.1.0/24. This
    gives us the range 10.0.1.0 to 10.0.1.255. This allows us to forward up to 256 services.
- Open a socket on one of the IP addresses in this range for each service that's being forwarded.
    Each port specified in a service will probably need to have a socket and a port-forward.
- Create a hosts file at `/etc/netns/[namespace-name]/hosts` mapping each service name to the
    corresponding IP address
    - Note that `/etc/netns/[namespace-name]/{hosts,nsswitch.conf,resolv.conf}` files are all bind
        mounted into the shell created by `ip netns exec`. I haven't been able to get the hosts
        file recognised yet though..
- Instead of the hosts file, it _might_ be necessary to:
  - Start a DNS server: https://github.com/bluejekyll/trust-dns#resolver that resolves each service
      name to the appropriate IP address.
  - Create a `resolv.conf` at `/etc/netns/[namespace-name]/resolv.conf` containing `nameserver
      [ip-address-of-our-hosted-nameserver]`.
- The following steps follow the article here: https://itnext.io/create-your-own-network-namespace-90aaebc745d
    but it's possible this is closer to what we want to achieve: https://serverfault.com/a/925349
    - Create a pair of virtual ethernet devices:
      ```sh
      ip link add veth0 type veth peer name veth1
      ```
    - Create the vnet0 network namespace
      ```sh
      ip netns add vnet0
      ```
    - Assign the veth0 interface to the vnet0 network namespace
      ```sh
      ip link set veth0 netns vnet0
      ```
    - Assign the 10.0.1.0/24 IP address range to the veth0 interface
      ```sh
      ip -n vnet0 addr add 10.0.1.0/24 dev veth0
      ```
    - Bring up the veth0 interface
      ```sh
      ip -n vnet0 link set veth0 up
      ```
    - Bring up the lo interface, because packets destined for 10.0.1.0/24 (like ping) goes through
      the "local" route table
      ```sh
      ip -n vnet0 link set lo up
      ```

- Clean up
  - all virtual ethernet devices
  - all `/etc/netns/[namespace-name]` directories (and perhaps the `/etc/netns` directory if we
      created it)
