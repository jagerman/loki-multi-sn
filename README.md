# Loki Multi SN Setup

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
node you want to run, assuming the box is only being used only for loki service nodes, you will
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

This package installs a loki-multi-sn debian/ubuntu package which installs systemd service templates
named `loki-node@.service`, `loki-storage-server@.service`, `lokinet-router@.service` as well as
systemd targets named `loki-nodes.target`, `loki-storage-servers.target`, and
`lokinet-routers.target`.

There will *also* be a lokid service node running on the default port (or you may have it already
configured and running).  It will use the basic, untemplated service files (`loki-node.service`,
`loki-storage-server.service`, and `lokinet-router.service`).

Alternatively you can mask these services before installing *any* of the loki debs to prevent them
from running; it'll work either way.  To mask them so that only the templated service nodes
described below get activated, run this command *before* installing lokid, loki-storage-server,
or this package:

    sudo systemctl mask loki-node.service loki-storage-server.service lokinet-router.service

Once masked, you can install using:

    sudo apt install loki-multi-sn

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
- storage server will listen on ports 221XX and 202XX (on the public IP)
- lokid p2p will listen on port 222XX (on the public IP)
- lokid quorumnet will listen on port 225XX (on the public IP)
- lokinet will listen on UDP port 109XX (on the public IP)

Internal (localhost) listeners:
- lokid rpc will listen on port 223XX
- lokid zmq rpc will listen on port 224XX
- lokinet rpc will listen on port 119XX
- the lokinet router will add and use local network 10.1XX.0.0/16 on a virtual interface named
  lokitunXX, on which the local snode is available at 10.1XX.0.1 (you probably don't need to worry
  about this).

As an example, let's say I have chosen the number 42.  Then my public IP will have services on ports
20242, 22142, 22242, 22542, and 11142; with internal (localhost) ports 22342 and 22442.  (If you are using
a firewall that blocks everything, add appropriate exceptions for *each* SN's five public ports).

### Step 2: Enable the service

Enable and start your service node cluster using the number you chose above (I'll continue using 42
as an example):

    sudo loki-multi-sn-create 42

This will set up basic configurations for the service node components, and enable and start these
services:

    loki-node@42.service
    loki-storage-server@42.service
    lokinet-router@42.service

If you want to control just one service, you use these templated names to manage the services.  For
example, to stop just this lokid:

    systemctl stop loki-node@42.service

or to view loki-storage-server log output:

    journalctl -u loki-storage-server@42 -af

The lokid data will be inside /var/lib/loki/node-42, the storage server data will be inside
/var/lib/loki/storage-42, and the lokinet data will be inside /var/lib/lokinet/router-42.
Configuration for each will be in /etc/loki/node-42.conf, /etc/loki/storage-42.conf, and
/etc/loki/lokinet-router-42.ini.

You will most likely want one additional piece to be able to query the service node: inside your
~/.bashrc add the following:

    for n in /etc/loki/node-*.conf; do
        p=${n/*node-/}
        p=${p/.conf/}
        alias lokid-$p="lokid --config=$n"
    done
    lokid_all() {
        for n in /etc/loki/node-*.conf; do
            p=${n/*node-/}
            p=${p/.conf/}
            echo -e "\nlokid-$p:"
            lokid --config=$n "$@"
        done
    }

When you log out and log in again you will now have a `lokid-42` alias that invokes commands on your
node 42, such as checking the status:

    $ lokid-42 status
    2019-12-29 02:22:36.107	I Loki 'Nimble Nerthus' (v6.1.0-1f61de91b)
    2019-12-29 02:22:36.107	I Generating SSL certificate
    Height: 434477/434477 (100.0%), net hash 54.78 MH/s, v6.1.0(net v13) (next fork in 10.9 days), up to date, 8(out)+11(in) connections, uptime 1d 6h 57m 4s
    SN: cc3427f59fb9ae3bf9ef00b411186575f2e4882c3e57b8f37087cb305fc54ca2 active, proof: 53.1 minutes ago, last pings: 2.7min (storage), 2.1min (lokinet)

and you will have a `lokid_all` command that runs a command on *all* the lokids, such as:

    $ lokid_all status

    lokid-00:
    2019-12-29 02:30:04.447	I Loki 'Nimble Nerthus' (v6.1.0-1f61de91b)
    2019-12-29 02:30:04.447	I Generating SSL certificate
    Height: 434484/434484 (100.0%), net hash 53.24 MH/s, v6.1.0(net v13) (next fork in 10.9 days), up to date, 9(out)+9(in) connections, uptime 1d 7h 4m 34s
    SN: 8c5c138501e37eadde733d46a24e0678be416d637cd692dbef22e837b45f6601 active, proof: 37 seconds ago, last pings: 8sec (storage), 4.8min (lokinet)

    lokid-01:
    2019-12-29 02:30:05.420	I Loki 'Nimble Nerthus' (v6.1.0-1f61de91b)
    2019-12-29 02:30:05.420	I Generating SSL certificate
    Height: 434484/434484 (100.0%), net hash 53.24 MH/s, v6.1.0(net v13) (next fork in 10.9 days), up to date, 8(out)+14(in) connections, uptime 1d 7h 4m 35s
    SN: 44e8e7ef21e9ff147b82f6409802adb682c2c91dee30b89f5199393866bc2a6c active, proof: 32 seconds ago, last pings: 9sec (storage), 4.9min (lokinet)

### Upgrades

A `loki-multi-sn-upgrade` script is installed that you should run after upgrading the loki-multi-sn
package.  It looks for upgrade-needed generated config files and upgrades them for you.  (When there
is nothing to upgrade it doesn't do anything, so always safe to run it, particularly for major
upgrades).

You need to be a bit more careful when updating.  When you upgrade one or more of the debs, only the
basic services (`loki-node.service`, `loki-storage-server.service`, and/or `lokinet-router.service`)
will be restart but *not* the templated versions, so you will need to do this manually.  One way is
to specify all the numbers, such as:

    sudo systemctl restart loki-node@01 loki-node@02 loki-node@03

which you can also shorten to:

    sudo systemctl restart loki-node@{01,02,03}

The templates services, however, also get a "target" which you can use:

    sudo systemctl restart loki-nodes.target

Similarly there are targets for storage server and routers: `loki-storage-servers.target` and
`lokinet-routers.target`.

Note, however, that targets only apply to currently running services, so if you have stopped some
you *cannot* use `sudo systemctl start loki-nodes.target` to start them all.

### Managing service node keys

You should keep a backup of your service node's private keys.  If your server node were to
irrecoverably crash or your ISP disconnects you, you will need them to set up your SN somewhere
else.

If you know how to properly make a copy of a binary file (e.g. using scp) then do it.  Otherwise,
one way you can back them up is to use the xxd tool:

    sudo apt install xxd

which lets you convert the binary keys into plain text data that you can easily copy and paste and
save somewhere.  For example:

    hades:~$ sudo xxd /var/lib/loki/node-01/key
    00000000: 627b 92a7 ee14 b25d d400 21e7 1a91 11bb  b{.....]..!.....
    00000010: 396d d2aa 4012 c73c 0743 e295 ffc2 9e0e  9m..@..<.C......

    hades:~$ sudo xxd /var/lib/loki/node-01/key_ed25519 
    00000000: db37 265a 1072 06db ff40 bab2 1a98 fd3d  .7&Z.r...@.....=
    00000010: 48ee 8af7 691d bed6 a5fe 776e e465 e082  H...i.....wn.e..
    00000020: 8193 5717 2977 af80 45ae df53 0cb5 fe69  ..W.)w..E..S...i
    00000030: d753 3c3e b32f ee2d adc7 0c04 8415 3822  .S<>./.-......8"

(I know, I know, the "Hades" hostname above is from the wrong pantheon, but I named this machine
long before Loki was born).

Copy and paste that content (the ed25519 key is supposed to be twice as long) and back it up
somewhere.  If you ever need to restore it you would use:

    xxd -r - /var/lib/loki/node-99/key

and then copy and paste the saved lines into the terminal then hit Ctrl-D.  This will result in an
exact binary copy, which is what you need.

### Faster syncing

Once you have one service node synced, you can start up another one much faster by stopping it and
copying its lmdb file to the new one.  Let's say I have service node 42 all synced and up and
running, and I want to create server node 77.

1. Set up service node 77 and let it run for 15 seconds, which should be long enough for it to
   create the initial lmdb database.

2. Stop both service node 42 and 77: `sudo systemctl stop loki-node@42 loki-node@77`

3. Copy 42's lmdb to 77's lmdb with:

       sudo cp /var/lib/loki/node-42/lmdb/data.mdb /var/lib/loki/node-77/lmdb/data.mdb

   Double check that you have this in the correct order!  The first file path should be the fully synced
   node, the second is the new one to overwrite.

4. Start both lokid's again with: `sudo systemctl start loki-node@42 loki-node@77`

5. Check on your lokid's with `lokid-42 status` and `lokid-77` status.  (This assumes you installed
   the bit of code in `~/.bashrc` that I mentioned earlier.  Also you will have to log out and in
   again before the `lokid-77` alias will work).
