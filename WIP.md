Notes
- https://news.ycombinator.com/item?id=30970720
- https://news.ycombinator.com/item?id=30983245
- https://github.com/rapiz1/rathole
- https://github.com/anderspitman/awesome-tunneling
- https://github.com/boringproxy/boringproxy
- https://github.com/wyhaya/updns
- https://github.com/EmilHernvall/dnsguide
- https://lib.rs/crates/tcp-warp
- Can we use https://github.com/bytecodealliance/cap-std or whatever is used for capabilities by
    https://github.com/cloudflare/boringtun ? See e.g. the `boringtun` command
    `sudo setcap cap_net_admin+epi boringtun` that sets the CAP_NET_ADMIN privilege specifically.
    (This comment is a bit mixed up, but should get future-me thinking..). The real important
    principle here is the principle of least privilege. We want the user to trust us. We don't want
    to install stuff in their cluster. We don't want them to have to use `sudo`, preferably even to
    assign privileges.
- Can we do a user-space proxy of some sort?
- Try to do this in user-space..
- One part of this is the local networking component, which will create a networking environment
    (either with a tunnel or a network namespace) with some sort of intercepted or controlled DNS
    resolution. This in itself is a useful thing to control, for example, if the system hosts file
    can't be modified (no root) or is a nuisance to modify (nix/guix). It's also rapidly
    reversible, by killing the process, more so than modifying the hosts file.
- C.f. the previous bullet about the controlled DNS resolution, we should enable custom DNS
    entries, or perhaps DNS entries corresponding to k8s ingresses, such that it's also possible to
    resolve services by their public hostnames. For example, if the user wants to access the
    application via hello.io, we could enable that. Or, similarly, if the k8s ingress says
    `hello.io` we should automatically resolve `hello.io` to the user's cluster.
- It could be useful to nest calls to our proxy, such that we can have a namespace that resolves
    `hello.io` to the user's cluster, then call our application again, but with a different
    cluster, such that calls to `example.com` resolve via our nested proxy to one cluster, and
    calls to `hello.io` resolve (via the "parent" namespace) to another cluster.
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
- Can we create a tunnel and just intercept certain DNS queries? What if the user uses DNS over
    HTTPS or DNS over TLS or DNSSEC? Or are those implemented at the network interface level? Might
    still need a DNS server..

Telepresence notes
- Creates a network device, without root:
    ```
    8: tel0: <POINTOPOINT,MULTICAST,NOARP,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UNKNOWN group default qlen 500
        link/none
        inet 10.245.0.0/16 scope global tel0
           valid_lft forever preferred_lft forever
        inet6 fe80::1ae0:860b:a670:40f9/64 scope link stable-privacy
           valid_lft forever preferred_lft forever
    ```
  I am in `groups`:
    ```
    $ groups
    users wheel dialout docker adbusers wireshark
    ```
- Using `strace`, it _looks_ like it's searching to see if we have sudo:
    ```
  newfstatat(AT_FDCWD, "/nix/store/8kgsjv57icc18qhpmj588g9x1w34hi4j-bash-interactive-5.1-p12/bin/sudo", 0xc000ae2518, 0) = -1 ENOENT (No such file or directory)
  newfstatat(AT_FDCWD, "/nix/store/9wjacdm5cy68ixhj6fd567hk5lldca30-patchelf-0.14.3/bin/sudo", 0xc000ae25e8, 0) = -1 ENOENT (No such file or directory)
  newfstatat(AT_FDCWD, "/nix/store/npm4g1zsj5yzygf6bq46pbi9fqhxisha-gcc-wrapper-10.3.0/bin/sudo", 0xc000ae26b8, 0) = -1 ENOENT (No such file or directory)
  newfstatat(AT_FDCWD, "/nix/store/mlfy9ifzszg7z2q6aiblvm5qkfn3bmwb-gcc-10.3.0/bin/sudo", 0xc000ae2788, 0) = -1 ENOENT (No such file or directory)
  newfstatat(AT_FDCWD, "/nix/store/4687f3vcym7a3ipjh0lfm5qlr12m76nr-glibc-2.33-59-bin/bin/sudo", 0xc000ae2858, 0) = -1 ENOENT (No such file or directory)
  newfstatat(AT_FDCWD, "/nix/store/176gh50y24c0lx2bnnmsvf9wazb73php-coreutils-9.0/bin/sudo", 0xc000ae2928, 0) = -1 ENOENT (No such file or directory)
  newfstatat(AT_FDCWD, "/nix/store/w327j7z9wlv7hym4spjzagax7c5hqvrf-binutils-wrapper-2.35.2/bin/sudo", 0xc000ae29f8, 0) = -1 ENOENT (No such file or directory)
  newfstatat(AT_FDCWD, "/nix/store/cdm6zywd51mbabxhklsixwcskv4n70s3-binutils-2.35.2/bin/sudo", 0xc000ae2ac8, 0) = -1 ENOENT (No such file or directory)
  newfstatat(AT_FDCWD, "/nix/store/vh482yvlkgdf4j6ixmnyyky3mf83zsfk-telepresence2-2.4.6/bin/sudo", 0xc000ae2b98, 0) = -1 ENOENT (No such file or directory)
  newfstatat(AT_FDCWD, "/nix/store/176gh50y24c0lx2bnnmsvf9wazb73php-coreutils-9.0/bin/sudo", 0xc000ae2c68, 0) = -1 ENOENT (No such file or directory)
  newfstatat(AT_FDCWD, "/nix/store/g0hhds9sdiqds0xxgpn7v4pwcv89varr-findutils-4.8.0/bin/sudo", 0xc000ae2d38, 0) = -1 ENOENT (No such file or directory)
  newfstatat(AT_FDCWD, "/nix/store/ywy7hggaj6vvgw86vbnyirll697ic5jx-diffutils-3.8/bin/sudo", 0xc000ae2e08, 0) = -1 ENOENT (No such file or directory)
  newfstatat(AT_FDCWD, "/nix/store/4na05j9gmpp3dwhmnc1q0a108ymf2qjy-gnused-4.8/bin/sudo", 0xc000ae2ed8, 0) = -1 ENOENT (No such file or directory)
  newfstatat(AT_FDCWD, "/nix/store/pkj79ap8x2dalzl63ndpdmxg2crxpjl8-gnugrep-3.7/bin/sudo", 0xc000ae2fa8, 0) = -1 ENOENT (No such file or directory)
  newfstatat(AT_FDCWD, "/nix/store/yabl1gmn7balb15hbcj613jcz0cxny42-gawk-5.1.1/bin/sudo", 0xc000ae3078, 0) = -1 ENOENT (No such file or directory)
  newfstatat(AT_FDCWD, "/nix/store/sdpqvnmahizaxbs3nnzmgfgyqsdxb1bw-gnutar-1.34/bin/sudo", 0xc000ae3148, 0) = -1 ENOENT (No such file or directory)
  newfstatat(AT_FDCWD, "/nix/store/7f3bipp5x4yiqghnkkv88rfsqzs6fl9z-gzip-1.11/bin/sudo", 0xc000ae3218, 0) = -1 ENOENT (No such file or directory)
  newfstatat(AT_FDCWD, "/nix/store/2ccy5zc89zpc2aznqxgvzp4wm1bwj05n-bzip2-1.0.6.0.2-bin/bin/sudo", 0xc000ae32e8, 0) = -1 ENOENT (No such file or directory)
  newfstatat(AT_FDCWD, "/nix/store/vhsx8rivchbvc6xnymyc45vvk7c7dz25-gnumake-4.3/bin/sudo", 0xc000ae33b8, 0) = -1 ENOENT (No such file or directory)
  newfstatat(AT_FDCWD, "/nix/store/pbfraw351mksnkp2ni9c4rkc9cpp89iv-bash-5.1-p12/bin/sudo", 0xc000ae3488, 0) = -1 ENOENT (No such file or directory)
  newfstatat(AT_FDCWD, "/nix/store/hjg2fnn41awrq6xzaq3cjfar6jwfcbw6-patch-2.7.6/bin/sudo", 0xc000ae3558, 0) = -1 ENOENT (No such file or directory)
  newfstatat(AT_FDCWD, "/nix/store/cxd7laqdidlqvqrlzsd4cls8470h6rwb-xz-5.2.5-bin/bin/sudo", 0xc000ae3628, 0) = -1 ENOENT (No such file or directory)
  newfstatat(AT_FDCWD, "/run/wrappers/bin/sudo", {st_mode=S_IFREG|S_ISUID|0511, st_size=17760, ...}, 0) = 0
    ```
- Routing table modified. The three routes for Iface tel0 were added:
    ```
    Kernel IP routing table
    Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
    default         _gateway        0.0.0.0         UG    2048   0        0 wlp59s0
    10.131.10.0     0.0.0.0         255.255.255.0   U     0      0        0 tel0
    10.244.0.0      0.0.0.0         255.255.255.0   U     0      0        0 tel0
    10.245.0.0      0.0.0.0         255.255.0.0     U     0      0        0 tel0
    172.17.0.0      0.0.0.0         255.255.0.0     U     0      0        0 docker0
    192.168.20.0    0.0.0.0         255.255.254.0   U     2048   0        0 wlp59s0
    _gateway        0.0.0.0         255.255.255.255 UH    2048   0        0 wlp59s0
    ```
- `iptables --list` was unchanged

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
