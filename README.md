# Session Multi SN Setup

The deb produced here gives you systemd service templates to help you run multiple service nodes on
one machine.

This is **NOT** officially supported, carries higher risk, and needs considerably more resources per
machine.

## Warning - Risk

Doing this is increasing your risk: if the machine goes down *all* the service nodes on it go down,
while if you use multiple VPSes the loss of one VPS affects only that one service node and not the
others.

## Warning - Hardware requirements

You also should take care to make sure the machine is capable of handling it.  For *each* service
node you want to run, assuming the box is only being used only for Oxen service nodes, you will
need the following (but note that these requirements are likely to increase in the future):

- 1/2 of a dedicated server high performance core (Ryzen or modern Intel Core).  A little bit more
  than half for a dedicated server with older CPUs.
- One *full* virtual CPU core on a VPS (because these are shared cores with other VPSes!)
- 2 GB of RAM
- 20 GB of free, SSD storage space (regular hard drives not recommended for blockchain storage).
- at least 100Mbps of bandwidth

For example, on a dedicated server with a 4 core (8 thread) Intel Xeon processor, 32 GB of RAM, and
180 GB of free hard drive space, and a 500Mbps connection you'd have a machine that has limits of:

8 SNs (CPU)
16 SNs (RAM)
6 SNs (storage)
5 SNs (bandwidth)

and so this machine could handle 5 SNs.  The same machine with a gigabit connection and 500GB of
storage would be good for 8 SNs (now limited by the CPU).

A second hypothetical example of a VPS that offers 4 virtual CPUs, 16GB RAM, 160GB disk space on a
1Gbps connection would be good for:

4 SNs (CPU)
8 SNs (RAM)
8 SNs (storage)
10 SNs (bandwidth)

so a limit of 4 service nodes.  (In my experience, each virtual CPU offered on a VPS is usually
substantially weaker than a single core of a dedicated server.  I'd be tempted to try load more
service nodes on the core of a dedicated server than I would to load more onto a "big" VPS.  So in
other words, a dedicated server with these same specifications but with real cores could easily
handle twice the CPU load of this VPS).

## Installing the package

Pre-built multi-service-node packages are available from the [Oxen apt repo](https://deb.oxen.io).

This package installs a oxen-multi-sn debian/ubuntu package which installs systemd service templates
named `oxen-node@.service`, `oxen-storage-server@.service`, `lokinet-router@.service` as well as
systemd targets named `oxen-nodes.target`, `oxen-storage-servers.target`, and
`lokinet-routers.target`.

There will *also* be a oxend service node running on the default port (or you may have it already
configured and running).  It will use the basic, untemplated service files (`oxen-node.service`,
`oxen-storage-server.service`, and `lokinet-router.service`).

Alternatively you can mask these services before installing *any* of the oxen debs to prevent them
from running; it'll work either way.  To mask them so that only the templated service nodes
described below get activated, run this command *before* installing oxend, oxen-storage-server,
or this package:

    sudo systemctl mask oxen-node.service oxen-storage-server.service lokinet-router.service

Once masked, you can install using:

    sudo apt install oxen-multi-sn

which will install them but, because they are masked, not start them.  (If you already have one
running with the basic debs then just don't unmask: you'll continue to have the default services
running *in addition* to the new templated onces we set up below).

## Setting up a service node

### Step 1: Choose a number

Choose a two-digit number from 00 to 99.  I recommend you start at 00 or 01 and then move up by one
for each one, but if you have some other system in mind go ahead as long as you use two-digit
numbers.

The number you choose will be used in the ports that your service nodes use: for service number with
number XX:

Public address listeners:
- storage server will listen on the public IP on ports 221XX (TCP) and 202XX (*both* TCP *and* UDP).
- oxend p2p will listen on port 222XX (on the public IP)
- oxend quorumnet will listen on port 225XX (on the public IP)
- lokinet will listen on UDP port 109XX (on the public IP)

Internal (localhost) listeners:
- oxend rpc will listen on port 223XX
- lokinet rpc will listen on port 119XX
- the lokinet router will add and use local network 10.1XX.0.0/16 on a virtual interface named
  lokitunXX, on which the local snode is available at 10.1XX.0.1 (you probably don't need to worry
  about this).

As an example, let's say I have chosen the number 42.  Then my public IP will have services on ports
20242 (TCP+UDP), 22142 (TCP), 22242 (TCP), 22542 (TCP), and 10942 (UDP); with internal (localhost)
ports 22342, 22442, and 11942.  (If you are using a firewall that blocks everything, add appropriate
exceptions for *each* SN's six [4 TCP, 2 UDP] public ports).

### Step 2: Enable the service

Enable and start your service node cluster using the number you chose above (I'll continue using 42
as an example):

    sudo oxen-multi-sn-create 42 https://l2.provider.example.org/rpc http://1.2.3.4/arb-rpc

where the arguments after "42" specify the l2-providers to use.  (You can also use oxend proxies;
run the script without any arguments for more usage info).

This will set up basic configurations for the service node components, and enable and start these
services:

    oxen-node@42.service
    oxen-storage-server@42.service
    lokinet-router@42.service

If you want to control just one service, you use these templated names to manage the services.  For
example, to stop just this oxend:

    systemctl stop oxen-node@42.service

or to view oxen-storage-server log output:

    journalctl -u oxen-storage-server@42 -af

The oxend data will be inside /var/lib/oxen/node-42, the storage server data will be inside
/var/lib/oxen/storage-42, and the lokinet data will be inside /var/lib/lokinet/router-42.
Configuration for each will be in /etc/oxen/node-42.conf, /etc/oxen/storage-42.conf, and
/etc/oxen/lokinet-router-42.ini.

You will most likely want one additional piece to be able to query the service node: inside your
~/.bashrc add the following:

    for n in /etc/oxen/node-*.conf; do
        p=${n/*node-/}
        p=${p/.conf/}
        alias oxend-$p="oxend --config=$n"
    done
    oxend_all() {
        for n in /etc/oxen/node-*.conf; do
            p=${n/*node-/}
            p=${p/.conf/}
            echo -e "\noxend-$p:"
            oxend --config=$n "$@"
        done
    }

When you log out and log in again you will now have a `oxend-42` alias that invokes commands on your
node 42, such as checking the status:

    $ oxend-42 status
    2019-12-29 02:22:36.107	I Loki 'Nimble Nerthus' (v6.1.0-1f61de91b)
    2019-12-29 02:22:36.107	I Generating SSL certificate
    Height: 434477/434477 (100.0%), net hash 54.78 MH/s, v6.1.0(net v13) (next fork in 10.9 days), up to date, 8(out)+11(in) connections, uptime 1d 6h 57m 4s
    SN: cc3427f59fb9ae3bf9ef00b411186575f2e4882c3e57b8f37087cb305fc54ca2 active, proof: 53.1 minutes ago, last pings: 2.7min (storage), 2.1min (lokinet)

and you will have a `oxend_all` command that runs a command on *all* the oxends, such as:

    $ oxend_all status

    oxend-00:
    2019-12-29 02:30:04.447	I Loki 'Nimble Nerthus' (v6.1.0-1f61de91b)
    2019-12-29 02:30:04.447	I Generating SSL certificate
    Height: 434484/434484 (100.0%), net hash 53.24 MH/s, v6.1.0(net v13) (next fork in 10.9 days), up to date, 9(out)+9(in) connections, uptime 1d 7h 4m 34s
    SN: 8c5c138501e37eadde733d46a24e0678be416d637cd692dbef22e837b45f6601 active, proof: 37 seconds ago, last pings: 8sec (storage), 4.8min (lokinet)

    oxend-01:
    2019-12-29 02:30:05.420	I Loki 'Nimble Nerthus' (v6.1.0-1f61de91b)
    2019-12-29 02:30:05.420	I Generating SSL certificate
    Height: 434484/434484 (100.0%), net hash 53.24 MH/s, v6.1.0(net v13) (next fork in 10.9 days), up to date, 8(out)+14(in) connections, uptime 1d 7h 4m 35s
    SN: 44e8e7ef21e9ff147b82f6409802adb682c2c91dee30b89f5199393866bc2a6c active, proof: 32 seconds ago, last pings: 9sec (storage), 4.9min (lokinet)

## Upgrades

A `oxen-multi-sn-upgrade` script is installed that you should run after upgrading the oxen-multi-sn
package.  It looks for upgrade-needed generated config files and upgrades them for you.  (When there
is nothing to upgrade it doesn't do anything, so always safe to run it, particularly for major
upgrades).

Generally you will be prompted during upgrade of the oxen-multi-sn package when such a manual
upgrade script run is needed.

As of the rebranded oxen debs, it is no longer necessary to manually restart after upgrading
the individual software components (oxend, oxen-storage-server, etc.).

Should you wish to restart manually anyway, you can use:

    sudo systemctl restart oxen-node@01 oxen-node@02 oxen-node@03

which you can also shorten to:

    sudo systemctl restart oxen-node@{01,02,03}

The templates services, however, also get a "target" which you can use:

    sudo systemctl restart oxen-nodes.target

Similarly there are targets for storage server and routers: `oxen-storage-servers.target` and
`lokinet-routers.target`.

Note, however, that targets only apply to currently running services, so if you have stopped some
you *cannot* use `sudo systemctl start oxen-nodes.target` to start them all: it will only restart
nodes that are already running.

## Managing service node keys

You should keep a backup of your service node's private keys.  If your server node were to
irrecoverably crash or your ISP disconnects you, you will need them to set up your SN somewhere
else.

If you know how to properly make a copy of a binary file (e.g. using scp) then back these files up
for whatever `NN` nodes you have set up:
- `/var/lib/oxen/node-NN/key_ed25519`
- `/var/lib/oxen/node-NN/key_bls` (new for Oxen 11!)
- if it exists, `/var/lib/oxen/node-NN/key`

Otherwise, a convenient way to back them up is to use the oxen-sn-keys tool (included with oxend)
which lets you convert the binary keys into plain text data that you can easily copy and paste and
save somewhere.  For example:

    hades:~$ sudo oxen-sn-keys show /var/lib/oxen/node-01/key
    /var/lib/oxen/node-01/key (legacy SN keypair)
    ==========
    Private key: f9d74c6cb83b1da9dae4df7400db545ba2bb61e9fb4b10f5e5b60dfcdd68cf04
    Public key:  cbcbb4d5527450d877420ccb90a3c3ae44b04da5913966f676c5ebc735f09ea1

(Note that the above `key` file will only exist on service nodes upgraded from loki 7.x or earlier;
new nodes created under loki/oxen 8.x and above will only have the following `key_ed25519` and
`key_bls` files, while earlier nodes will have all three. To restore a service node you always need
`key_ed25519` and `key_bls, and need `key` if it exists).

    hades:~$ sudo oxen-sn-keys show /var/lib/oxen/node-01/key_ed25519 
    /var/lib/oxen/node-01/key_ed25519 (Ed25519 SN keypair)
    ==========
    Secret key:      dacf1adeed1d7d2821f77ac13015493f4459c8c4d83334e9456a5b04993599da
    Public key:      16be3acc80150c8f0fa97ffa3bdbfb2a3927f570a553ecb2acf018b20891956b
    X25519 pubkey:   bcf303f32526687a82bd4279d2e1f638f6b5347e84a8cb9f46bc187ebfc44c29
    Lokinet address: n49diurynwge6d7jx97dzs95feh1x7mowij63cic6ycmrnrt1iio.snode

    hades:~$ sudo oxen-sn-keys show /var/lib/oxen/node-01/key_bls
    /var/lib/oxen/node-01/key_bls (BLS SN keypair)
    ==========
    Secret key:      0x20cde78c2608259d88a4142287c392ea0bbd8ceea8f837eccaaec1aa1c8f38f9
    Public key:      0x1abc39c9dc4de40f92c3cb2b3e74e889c139883b1bb627408e948d742a2a5d021c406bbbe7961bad22636a40aefbb1ed1ae4a7e2b74170b7d7ab2548b0274eee

Copy and paste those outputs and back them up somewhere.  If you ever need to restore them you would
use:

    sudo oxen-sn-keys restore /var/lib/oxen/node-99/key_ed25519
    sudo oxen-sn-keys restore-bls /var/lib/oxen/node-99/key_bls

And also, if you had the older legacy SN keypair `key` file:

    sudo oxen-sn-keys restore-legacy /var/lib/oxen/node-99/key

These commands will prompt you for the key info displayed by the `show` commands.

## Faster syncing

Once you have one service node synced, you can start up another one much faster by stopping it and
copying its lmdb file to the new one.  Let's say I have service node 42 all synced and up and
running, and I want to create server node 77.

1. Set up service node 77 and let it run for 15 seconds, which should be long enough for it to
   create the initial lmdb database.

2. Stop both service node 42 and 77: `sudo systemctl stop oxen-node@42 oxen-node@77`

3. Copy 42's lmdb to 77's lmdb with:

       sudo cp -p /var/lib/oxen/node-42/lmdb/data.mdb /var/lib/oxen/node-77/lmdb/data.mdb

   Double check that you have this in the correct order!  The first file path should be the fully synced
   node, the second is the new one to overwrite.

4. Start both oxend's again with: `sudo systemctl start oxen-node@42 oxen-node@77`

5. Check on your oxend's with `oxend-42 status` and `oxend-77` status.  (This assumes you installed
   the bit of code in `~/.bashrc` that I mentioned earlier.  Also you will have to log out and in
   again before the `oxend-77` alias will work).

## Using oxend as an L2 proxy

When running multiple service nodes -- whether or not on the same machine -- it is useful to be able
to designate just one or two as the nodes that talk to the actual L2 provider, and have the others
talk to that L2-provider-connected oxend in "L2 proxy" mode where that node can provide L2
information to the other nodes.

See the [oxend L2 proxy docs
page](https://docs.getsession.org/user-guides/session-stagenet-node-setup/how-to-set-up-an-oxend-l2-proxy)
for more details on how this works.

The scripts in this package can update or create oxen "NN" nodes that use an L2 proxy (instead of a
full L2 provider), but the proxy itself must still be configured manually.  See the docs page linked
above for the overall details.  For upgrading, see the section below.  For setting up a new server,
a quick recipe for setting up the "00" node on a server as the l2-provider, with the "01" through
"07" nodes on the same machine using the "00" node as the l2 proxy, is as follows:

0. Obtain access to an Arbitrum One L2 RPC provider.  Ideally obtain access to two different
   providers, so that you can fall back to another provider in case of connectivity issues to the
   first provider.  I will use "https://example.org/arb" and "https://backup.example.com/arb" as my
   primary and backup providers in this example.
1. Set up the "00" oxen-node using `oxen-multi-sn-create 00 https://example.org/arb
   https://backup.example.com/arb`.  (If the 00 node already exists, double-check that its
   `l2-provider=` values in `/etc/oxen/node-00.conf` match what you think they should be).
2. Reconfigure the 00 node to act as a proxy by editing `/etc/oxen/node-00.conf` and adding the
   line:

       l2-proxy=/etc/oxen/proxy-pubkeys.txt

3. Create the above file using `touch /etc/oxen/proxy-pubkeys.txt` (we don't need to put anything
   in it yet).
4. Restart node 00: `systemctl restart oxen-node@00`.
5. Set up the "01" oxen-node using `oxen-multi-sn-create 01 l2-oxend ipc:///var/lib/oxen/node-00/oxend.sock`
6. Repeat step 4 for 02 through 07.  (Note that "01" changes, but the "node-00" part doesn't).

This will result in a setup where the oxen-node@00 is responsible for providing L2 information to
all the other nodes on the same server.

If you want to extend this proxy access to service nodes running on remote servers then you can set
this up as follows:

7. For each remote oxend that you want to be able to connect, obtain its pubkey and add it into the
   /etc/oxen/proxy-pubkeys.txt file on the proxy server.  (You do *not* have to restart the
   oxen-node@00 oxend after changing this file; it will detect changes and reload it).
8. Make a note of the oxen-node@00's public IP, its quorumnet port (225NN, e.g. 22500 for
   oxen-node@00), and its pubkey.
   - For steps 7 and 8, if you are following these instructions using a very old pre-Oxen 8 node
     with separate primary and Ed25519 pubkeys, you must use the Ed25519 pubkeys!
9. When configuring the remote, for a non-multi-sn setup, specify "l2-oxend=IP:PORT/PUBKEY".  For a
   multi-sn setup, run `oxen-multi-sn-create NN l2-oxend IP:PORT/PUBKEY` for a new node, or
   `oxen-multi-sn-upgrade l2-oxend IP:PORT/PUBKEY` when first upgrading a multi-sn server to the
   11.2+ release.

### Upgrading an existing oxen-multi-sn setup

If you already have a oxen-multi-sn setup (e.g. upgrading from oxen 10), you will want to follow
steps 0-4 above to get one of your existing nodes working as a proxy with an configured l2-provider.
I'll continue to assume that it is node 00, but it could be any of them.  The rest of the nodes can
then be upgraded using:

    oxen-multi-sn-upgrade l2-oxend ipc:///var/lib/oxen/node-00/oxend.sock

(be sure to change 00 to whichever node you are using).  This will add the l2-oxend= config line to
all the other files at once.

You can also set up multiple nodes at once.  For instance if you wanted to use my 00 node *and* a
remote node for redundancy, then you would follow steps 7-8 above, and then specify both the ipc for
node-00, and the remote address for the other proxy, like this:

    oxen-multi-sn-upgrade l2-oxend ipc:///var/lib/oxen/node-00/oxend.sock IP:PORT/PUBKEY

This command only does something to node-NN configs that do not have any L2 configuration at all: if
you want to edit it later then you need to edit the /etc/oxen/node-NN.conf files to add the relevant
l2-oxend= lines.
