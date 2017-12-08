Mirage OS hackaton December 2017
=================================

Personal notes on MirageOS, Ocaml ecosystem and charrua-core (Ocaml DHCP client).

Thanks so much [@hannesm](https://github.com/hannesm) for organizing it.

Install ``ocaml``, ``opam`` and ``mirage`` in Debian testing/sid
----------------------------------------------

Following [Mirage Install](https://mirage.io/wiki/install):

    apt install install ocaml ocaml-native-compilers camlp4-extra opam
    opam init
    eval `opam config env`

    opam install mirage

Install ``charrua-core`` (DHCP client) local branch
------------------------------------------------


    git clone https://github.com/mirage/charrua-core
    cd charrua-core
    git checkout -b mybranch
    opam pin add charrua-core .

The installed code would be now in ``$HOME/.opam``

Use ``charrue-core`` in ``utop``
---------------------------------

Use charrue-core in ``utop`` (with help from [@cfcs](https://github.com/cfcs))

    utop
    #load "/home//user/.opam/system/lib/io-page-unix/io_page_unix.cma";;
    #require "charrua-core";;
    #require "charrua-core.wire";;
    open Dhcp_wire;;
    Dhcp_wire.ABSOLUTE_TIME;;
    ABSOLUTE_TIME;;


Ocaml/Python equivalences
-------------------------

    open Dhcp_wire;;
    
equivalent to:

    import * from DHCP_wire


Run charrua client in unix
------------------------------

DHCP client (which doesn't set an IP) by [@yomimono](https://github.com/yomimono):

    git clone https://github.com/yomimono/charrua-core 
    git checkout unix-charruac
    jbuilder build -p charrua-unix
    
    sudo _build/default/unix/client/charruac.exe

In case it's needed to rebuild dependencies:

    jbuilder clean -p charrua-client
    jbuilder clean -p charrua-client-lwt
    jbuilder clean -p charrua-core
    jbuilder build --dev @install -p charrua-unix

    sudo _build/default/unix/client/charruac.exe

To use last master version of rawlink (thanks to [@yomimono](https://github.com/yomimono) and [@haesbaert](https://github.com/haesbaert)):

    git clone https://github.com/haesbaert/rawlink
    cd rawlink
    opam pin add rawlink ./

Create ``solo5`` unikernel with ``charrua-core``
------------------------------------------------

(With help from [@mato](https://github.com/mato))

When there's a ``configure.ml`` in the module

    mirage configure -t unix
    make depend
    make

When there isn't.
Create a tun/tap interface:

    ip tuntap add tap100 mode tap
    ip addr add 10.0.0.1/24 dev tap100
    ip link set dev tap100 up


Then bridge to the tun/tap interface, in ``/etc/network/interfaces``:

    # Host-only bridge
    iface hostbr0 inet manual
        up ip link add hostbr0-master address 02:00:00:00:00:01 type dummy
        up ip link set dev hostbr0-master up
        up ip link add hostbr0 type bridge
        up ip link set dev hostbr0-master master hostbr0
        up ip addr add 10.0.120.1/24 dev hostbr0
        up ip link set dev hostbr0 up
        post-up /usr/sbin/dnsmasq -9RhK -i hostbr0 -F 10.0.120.128,10.0.120.254 -p 0 -x /var/run/dnsmasq-hostbr0.pid
        pre-down kill $(cat /var/run/dnsmasq-hostbr0.pid)
        down ip link del hostbr0
        down ip link del hostbr0-master



### With an IP

    cd mirage-skeleton/device-usage/network
    opam switch
    opam list
    mirage configure -t ukvm
    make depend
    make
    ./ukvm-bin --net=tap100 ./network.ukvm --ipv4=10.0.10.2/24 --logs=\*:debug

### With DHCP

    cd mirage-skeleton/device-usage/network
    rm mirage-unikernel-network-ukvm.opam 
    mirage configure -t ukvm --DHCP=true
    make depend
    make
    ./ukvm-bin --net=tap100 ./network.ukvm  --logs=\*:debug 


To use the local ``charrua-core`` git branch:

    cd charrua-core 
    opam pin add charrua-client-mirage ./

    cd mirage-skeleton/device-usage/network
    mirage clean
    mirage configure -t ukvm --DHCP=true
    make
    ./ukvm-bin --net=tap100 ./network.ukvm  --logs=\*:debug 

Execute an script from Ocaml
-----------------------------

(Helped by [@reynir](https://github.com/reynir) and [@yomimono](https://github.com/yomimono))
Using [Bos library](http://erratique.ch/software/bos) 

Without envirnoment variables:

    #require "bos";;
    open Bos;;
    OS.Cmd.to_null (OS.Cmd.run_io Cmd.(Cmd.v "mplayer" % "example.ogg") OS.Cmd.in_null);;

Passing envirnoment variables:

    let myenv = Astring.String.Map.empty;;
    let myenv = Astring.String.Map.add "reason" "foo" myenv;;
    OS.Cmd.to_null (OS.Cmd.run_io ~env:myenv Cmd.(Cmd.v "/sbin/dhclient-script") OS.Cmd.in_null);;

To see the environment variables passed to the script:

    let myres = OS.Cmd.to_string (OS.Cmd.run_io ~env:myenv Cmd.(Cmd.v "/usr/bin/env") OS.Cmd.in_null);;
    let mystr = match myres with
      | Result.Ok anything -> anything;;
    print_endline letmystr;;
