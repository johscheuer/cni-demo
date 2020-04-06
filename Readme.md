# CNI demo

In order to use this demo you need [vargant](https://www.vagrantup.com/) or a Linux machines.
Start the demo setup with `vagrant up`.

## Setup requirements

Install runc:

```bash
curl -sLO https://github.com/opencontainers/runc/releases/download/v1.0.0-rc10/runc.amd64
chmod +x runc.amd64
sudo mv runc.amd64 /usr/local/bin/runc
```

Install CNI:

```bash
curl -sLO https://github.com/containernetworking/plugins/releases/download/v0.8.5/cni-plugins-linux-amd64-v0.8.5.tgz
tar xvfz cni-plugins-linux-amd64-v0.8.5.tgz
sudo mkdir -p /opt/cni/bin/
sudo cp {bridge,host-local} /opt/cni/bin/
sudo chmod +x /opt/cni/bin/{bridge,host-local}
```

Prepare the interface:

```bash
sudo ip link add name cbr0 type bridge
sudo ip addr add 192.168.199.1/24 dev cbr0
sudo ip addr add fde4:8dba:82e1::1/64 dev cbr0
sudo ip link set dev cbr0 up
```

Create the configuration for CNI:

```bash
sudo mkdir -p /opt/cni/netconfs/
sudo tee /opt/cni/netconfs/10-connet.conf <<EOF
{
    "cniVersion": "0.4.0",
    "name": "connet",
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
            { "dst": "0.0.0.0/0" },
            { "dst": "fde4:8dba:82e1::1/64" }
        ],
     "dataDir": "/run/ipam-state"
    },
    "dns": {
      "nameservers": [ "8.8.8.8" ]
    }
}
EOF
```

In order to start a container we need an image, since runc is "only" a container runtime we need to prepull the image an export it;
If you have docker installed on your system you can use `docker export`, for this demo we will use [img](https://github.com/genuinetools/img):

```bash
curl -sLO https://github.com/genuinetools/img/releases/download/v0.5.7/img-linux-amd64
chmod +x img-linux-amd64
sudo mv img-linux-amd64 /usr/local/bin/img
```

Now we can fetch an image and unpack it:

```bash
img pull debian:buster
mkdir demo && cd demo
img unpack debian:buster
```

This command will unpack the root filesystem of the image into `rootfs`.
With the rootfs we can use `runc` to start our container:

```bash
# This will create the runtime config
runc spec --rootless
```

Inspect the `config.json` to see which namespaces should be created for this container:

```bash
$ sudo apt update && sudo apt install -y jq
# Show the config
$ jq '.linux.namespaces' config.json
[
  {
    "type": "pid"
  },
  {
    "type": "ipc"
  },
  {
    "type": "uts"
  },
  {
    "type": "mount"
  },
  {
    "type": "user"
  }
]
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
$ runc create demo
$ runc list
ID          PID         STATUS      BUNDLE               CREATED                          OWNER
demo        2348        created     /home/vagrant/demo   2020-04-03T07:22:21.496780818Z   vagrant
```

With thw following command we can jump into the container and show the network interfaces:

```bash
runc exec -t demo ip -o a s
1: lo    inet 127.0.0.1/8 scope host lo\       valid_lft forever preferred_lft forever
1: lo    inet6 ::1/128 scope host \       valid_lft forever preferred_lft forever
```

Now we fetch the network namespace of the container:

```bash
netns=$(jq -r '.namespace_paths.NEWNET' /var/run/user/1000/runc/demo/state.json)
```

In order to make `ip netns` work we can create a symlink:

```bash
sudo mkdir -p /var/run/netns
sudo -E ln -s $netns /var/run/netns/demo
```

Finally we can setup the network with CNI:

```bash
# We set some CNI specific environment variables
export CNI_PATH=/opt/cni/bin/
export CNI_CONTAINERID=demo
export CNI_NETNS=/var/run/netns/demo
export CNI_IFNAME=eth0
export CNI_COMMAND=ADD
$ cat /opt/cni/netconfs/10-connet.conf | sudo -E $CNI_PATH/bridge
{
    "cniVersion": "0.4.0",
    "interfaces": [
        {
            "name": "cbr0",
            "mac": "e6:60:4c:dd:76:7e"
        },
        {
            "name": "vethdc9adb78",
            "mac": "e6:60:4c:dd:76:7e"
        },
        {
            "name": "eth0",
            "mac": "4e:f7:e9:09:62:0f",
            "sandbox": "/var/run/netns/demo"
        }
    ],
    "ips": [
        {
            "version": "4",
            "interface": 2,
            "address": "192.168.199.2/24",
            "gateway": "192.168.199.1"
        },
        {
            "version": "6",
            "interface": 2,
            "address": "fde4:8dba:82e1::2/64",
            "gateway": "fde4:8dba:82e1::1"
        }
    ],
    "routes": [
        {
            "dst": "0.0.0.0/0"
        },
        {
            "dst": "fde4:8dba:82e1::1/64"
        }
    ],
    "dns": {
        "nameservers": [
            "8.8.8.8"
        ]
    }
}
```

and we can verify that the interface inside the container was created:

```bash
$ sudo ip netns exec demo ip -o a s eth0
3: eth0    inet 192.168.199.2/24 brd 192.168.199.255 scope global eth0\       valid_lft forever preferred_lft forever
3: eth0    inet6 fde4:8dba:82e1::2/64 scope global \       valid_lft forever preferred_lft forever
3: eth0    inet6 fe80::4cf7:e9ff:fe09:620f/64 scope link \       valid_lft forever preferred_lft forever
$ sudo ip netns exec demo ip route
default via 192.168.199.1 dev eth0
192.168.199.0/24 dev eth0 proto kernel scope link src 192.168.199.2
```

and we are able to ping the container from the host:

```bash
$ ping6 -c 3 fde4:8dba:82e1::2
PING fde4:8dba:82e1::2(fde4:8dba:82e1::2) 56 data bytes
64 bytes from fde4:8dba:82e1::2: icmp_seq=1 ttl=64 time=0.135 ms
64 bytes from fde4:8dba:82e1::2: icmp_seq=2 ttl=64 time=0.049 ms
64 bytes from fde4:8dba:82e1::2: icmp_seq=3 ttl=64 time=0.090 ms

--- fde4:8dba:82e1::2 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2042ms
rtt min/avg/max/mdev = 0.049/0.091/0.135/0.035 ms
```

and also with IPv4:

```bash
ping -c 3 192.168.199.2
PING 192.168.199.2 (192.168.199.2) 56(84) bytes of data.
64 bytes from 192.168.199.2: icmp_seq=1 ttl=64 time=0.044 ms
64 bytes from 192.168.199.2: icmp_seq=2 ttl=64 time=0.050 ms
64 bytes from 192.168.199.2: icmp_seq=3 ttl=64 time=0.047 ms

--- 192.168.199.2 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2035ms
rtt min/avg/max/mdev = 0.044/0.047/0.050/0.002 ms
```

we can also delete the interface again:

```bash
export CNI_PATH=/opt/cni/bin/
export CNI_CONTAINERID=demo
export CNI_NETNS=/var/run/netns/demo
export CNI_IFNAME=eth0
export CNI_COMMAND=DEL
$ cat /opt/cni/netconfs/10-connet.conf | sudo -E $CNI_PATH/bridge
# There is no output expected. When you see an output you ran into an error
```

and a last check:

```bash
sudo ip netns exec demo ip -o a s eth0
Device "eth0" does not exist.
```
