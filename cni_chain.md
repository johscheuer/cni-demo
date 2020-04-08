# CNI chaining

In the next part we want to take a look at CNI chaining.

## Setup the chain

At first we need to move a further cni plugin into the cni bin folder:

```bash
sudo mv $HOME/bandwidth /opt/cni/bin/
```

The [bandwidth](https://github.com/containernetworking/plugins/tree/master/plugins/meta/bandwidth) can be used to limited the egress and ingress bandwidth of an container.

We need to adjust our previous config:

```bash
sudo rm /opt/cni/netconfs/10-connet.conf
sudo tee /opt/cni/netconfs/10-connet.conflist <<EOF
{
  "cniVersion": "0.4.0",
  "name": "connet",
  "plugins": [
    {
      "type": "bridge",
      "bridge": "cbr0",
      "isGateway": true,
      "ipMasq": true,
      "ipam": {
        "type": "host-local",
        "ranges": [
          [
            {
              "subnet": "192.168.199.0/24"
            }
          ],
          [
            {
              "subnet": "fde4:8dba:82e1::/64"
            }
          ]
        ],
        "routes": [
          {
            "dst": "0.0.0.0/0"
          },
          {
            "dst": "fde4:8dba:82e1::1/64"
          }
        ],
        "dataDir": "/run/ipam-state"
      },
      "dns": {
        "nameservers": [
          "8.8.8.8"
        ]
      }
    },
    {
      "type": "bandwidth",
      "capabilities": {"bandwidth": true}
    }
  ]
}
EOF
```

Maybe you noticed that the plugins are now wrapped inside the `plugins` keyword. This list describes which plugins should be called.

Now we can reproduce the steps from before to create a container:

```bash
img pull networkstatic/iperf3
mkdir demo-chain && cd demo-chain
img unpack networkstatic/iperf3
```

This command will unpack the root filesystem of the image into `rootfs`.
With the rootfs we can use `runc` to start our container:

```bash
# This will create the runtime config
runc spec --rootless
```

We can see that the `network` namespace is missing so we need to add it:

```bash
# Add the network namespace
jq '.linux.namespaces += [input]' config.json <(echo '{"type":"network"}') > ./config.json.tmp
mv config.json.tmp config.json
```

and we also set the `terminal` parameter to `false`:

```bash
jq '.process.terminal=false' config.json > config.json.tmp
mv config.json.tmp config.json
```

Now we can actually start the container:

```bash
$ runc create client
$ runc create server
$ runc list
ID          PID         STATUS      BUNDLE                     CREATED                          OWNER
client      4363        created     /home/vagrant/demo-chain   2020-04-07T13:39:16.124629345Z   vagrant
server      4377        created     /home/vagrant/demo-chain   2020-04-07T13:39:20.333367024Z   vagrant
```

Let's setup the symlinks again:

```bash
for inst in "server" "client"; do
    netns=$(jq -r '.namespace_paths.NEWNET' /var/run/user/1000/runc/$inst/state.json)
    sudo -E ln -s $netns /var/run/netns/$inst
done
```

Before we can setup the network we need to install the [cnitool](https://github.com/containernetworking/cni/tree/master/cnitool):

```bash
# Install golang
sudo snap install go  --classic
# Install the cnitool
go get github.com/containernetworking/cni
go install github.com/containernetworking/cni/cnitool
# Make the cnitool binary available
sudo mv /home/vagrant/go/bin/cnitool /usr/local/bin/
```

```bash
# Actually this would come from the container runtime e.g. containerd
export CAP_ARGS='{
  "bandwidth": {
    "ingressRate": 1073741824,
    "ingressBurst": 1073741824,
    "egressRate": 1073741824,
    "egressBurst": 1073741824
  }
}'
# We set some CNI specific environment variables
export CNI_PATH=/opt/cni/bin/
export NETCONFPATH=/opt/cni/netconfs
$ sudo -E cnitool add connet /var/run/netns/server
...
$ sudo -E cnitool add connet /var/run/netns/client
```

The `cnitool` can also be used without the cni chaining.
And we can verify that the interface inside the container was created:

```bash
$ sudo ip netns exec server ip -o a s eth0
5: eth0    inet 192.168.199.9/24 brd 192.168.199.255 scope global eth0\       valid_lft forever preferred_lft forever
5: eth0    inet6 fde4:8dba:82e1::9/64 scope global \       valid_lft forever preferred_lft forever
5: eth0    inet6 fe80::8837:e6ff:fe33:78fb/64 scope link \       valid_lft forever preferred_lft forever
$  sudo ip netns exec client ip -o a s eth0
5: eth0    inet 192.168.199.10/24 brd 192.168.199.255 scope global eth0\       valid_lft forever preferred_lft forever
5: eth0    inet6 fde4:8dba:82e1::a/64 scope global \       valid_lft forever preferred_lft forever
5: eth0    inet6 fe80::80d7:3bff:fe8b:c440/64 scope link \       valid_lft forever preferred_lft forever
```

Now we need two terminals (either with `tmux` or separate terminals):

```bash
runc exec -t server iperf3 -s
```

and in the second terminal:

```bash
$ runc exec -t client iperf3 -c 192.168.199.9
Connecting to host 192.168.199.9, port 5201
[  4] local 192.168.199.12 port 36922 connected to 192.168.199.9 port 5201
[ ID] Interval           Transfer     Bandwidth       Retr  Cwnd
[  4]   0.00-1.00   sec   248 MBytes  2.08 Gbits/sec    0   3.01 MBytes
[  4]   1.00-2.00   sec   122 MBytes  1.03 Gbits/sec    0   3.01 MBytes
[  4]   2.00-3.00   sec   122 MBytes  1.03 Gbits/sec    0   3.01 MBytes
[  4]   3.00-4.00   sec   122 MBytes  1.03 Gbits/sec    0   3.01 MBytes
[  4]   4.00-5.00   sec   122 MBytes  1.03 Gbits/sec    0   3.01 MBytes
[  4]   5.00-6.00   sec   122 MBytes  1.03 Gbits/sec    0   3.01 MBytes
[  4]   6.00-7.00   sec   122 MBytes  1.03 Gbits/sec    0   3.01 MBytes
[  4]   7.00-8.00   sec   122 MBytes  1.03 Gbits/sec    0   3.01 MBytes
[  4]   8.00-9.00   sec   122 MBytes  1.03 Gbits/sec    0   3.01 MBytes
[  4]   9.00-10.00  sec   122 MBytes  1.03 Gbits/sec    0   3.01 MBytes
- - - - - - - - - - - - - - - - - - - - - - - - -
[ ID] Interval           Transfer     Bandwidth       Retr
[  4]   0.00-10.00  sec  1.32 GBytes  1.13 Gbits/sec    0             sender
[  4]   0.00-10.00  sec  1.32 GBytes  1.13 Gbits/sec                  receiver

iperf Done.
```

we can also validate that it works for the IPv6 address:

```bash
iperf3 -6 -c fde4:8dba:82e1::b
Connecting to host fde4:8dba:82e1::9, port 5201
[  4] local fde4:8dba:82e1::c port 47850 connected to fde4:8dba:82e1::9 port 5201
[ ID] Interval           Transfer     Bandwidth       Retr  Cwnd
[  4]   0.00-1.00   sec   245 MBytes  2.05 Gbits/sec    1   3.02 MBytes
[  4]   1.00-2.01   sec   116 MBytes   967 Mbits/sec    2   3.02 MBytes
[  4]   2.01-3.13   sec   110 MBytes   821 Mbits/sec    1   3.02 MBytes
[  4]   3.13-4.00   sec   125 MBytes  1.21 Gbits/sec    1   3.02 MBytes
[  4]   4.00-5.00   sec   131 MBytes  1.10 Gbits/sec    1   3.02 MBytes
[  4]   5.00-6.00   sec   121 MBytes  1.02 Gbits/sec    0   3.02 MBytes
[  4]   6.00-7.00   sec   120 MBytes  1.01 Gbits/sec    0   3.02 MBytes
[  4]   7.00-8.00   sec   121 MBytes  1.02 Gbits/sec    0   3.02 MBytes
[  4]   8.00-9.00   sec   120 MBytes  1.01 Gbits/sec    0   3.02 MBytes
[  4]   9.00-10.00  sec   121 MBytes  1.02 Gbits/sec    0   3.02 MBytes
- - - - - - - - - - - - - - - - - - - - - - - - -
[ ID] Interval           Transfer     Bandwidth       Retr
[  4]   0.00-10.00  sec  1.30 GBytes  1.12 Gbits/sec    6             sender
[  4]   0.00-10.00  sec  1.30 GBytes  1.12 Gbits/sec                  receiver

iperf Done.
```

If we ran the same command without the bandwidth limitation we get these vaules (depending on your CPU):

```bash
runc exec -t client iperf3 -c 192.168.199.13
Connecting to host 192.168.199.13, port 5201
[  4] local 192.168.199.14 port 60566 connected to 192.168.199.13 port 5201
[ ID] Interval           Transfer     Bandwidth       Retr  Cwnd
[  4]   0.00-1.00   sec  8.58 GBytes  73.7 Gbits/sec    0   2.03 MBytes
[  4]   1.00-2.00   sec  8.76 GBytes  75.3 Gbits/sec    1   2.03 MBytes
[  4]   2.00-3.00   sec  7.85 GBytes  67.4 Gbits/sec    1   2.04 MBytes
[  4]   3.00-4.00   sec  6.51 GBytes  55.9 Gbits/sec    3   2.05 MBytes
[  4]   4.00-5.00   sec  5.82 GBytes  50.0 Gbits/sec    1   2.07 MBytes
[  4]   5.00-6.00   sec  8.38 GBytes  71.9 Gbits/sec    0   2.07 MBytes
[  4]   6.00-7.00   sec  7.80 GBytes  67.0 Gbits/sec    1   2.09 MBytes
[  4]   7.00-8.00   sec  8.66 GBytes  74.4 Gbits/sec    0   2.09 MBytes
[  4]   8.00-9.00   sec  8.61 GBytes  74.0 Gbits/sec    0   2.09 MBytes
[  4]   9.00-10.00  sec  8.67 GBytes  74.4 Gbits/sec    0   2.10 MBytes
- - - - - - - - - - - - - - - - - - - - - - - - -
[ ID] Interval           Transfer     Bandwidth       Retr
[  4]   0.00-10.00  sec  79.6 GBytes  68.4 Gbits/sec    7             sender
[  4]   0.00-10.00  sec  79.6 GBytes  68.4 Gbits/sec                  receiver

iperf Done.
```

Clean up:

```bash
for inst in "server" "client"; do
    sudo -E cnitool del connet /var/run/netns/server$inst
    sudo rm /var/run/netns/$inst
    runc delete $inst
done
```
