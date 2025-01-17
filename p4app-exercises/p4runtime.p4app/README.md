# Implementing a Control Plane using P4Runtime

## Introduction

In this exercise, we will be building on the same P4
program that you used in the [basic_tunnel](../basic_tunnel.p4app) exercise. The
P4 program has been renamed to `advanced_tunnel.py` and has been augmented
with two counters (`ingressTunnelCounter`, `egressTunnelCounter`) and
two new actions (`myTunnel_ingress`, `myTunnel_egress`), which take care of
performing encap and decap on IPv4 packets using our tunneling protocol.

Like the previous exercises, running the `make` command creates a new
docker container and runs the program `main.py`, which compiles the P4
program and starts Mininet.  Unlike the previous exercises, `main.py`
does not install any table entries into the switches.  In this
exercise, you will use the program `mycontroller.py` to create the
table entries necessary to tunnel traffic between host 1 and 2.  Since
the exercise can be completed by making changes only to
`mycontroller.py`, and no changes to the P4 program are required, you
may choose to run `make` once in one window, and leave that running
for the entire exercise. In a separate terminal, you can quit
`mycontroller.py`, edit it, and start it again, and the new controller
process will take over control of the existing Mininet network.

> **Spoiler alert:** There is a reference solution in the `solution`
> sub-directory. Feel free to compare your implementation to the
> reference.

## Step 1: Run the (incomplete) starter code

The starter code for this assignment is in a file called `mycontroller.py`,
and it will install only some of the rules that you need to tunnel traffic between
two hosts.

Let's first compile the new P4 program, start the network, use `mycontroller.py`
to install a few rules, and look at the `ingressTunnelCounter` to see that things
are working as expected.

1. In your shell, run:
   ```bash
   make
   ```
   This will:
   * compile `advanced_tunnel.p4`,
   * start a Mininet instance with three switches (`s1`, `s2`, `s3`)
     configured in a triangle, each connected to one host (`h1`, `h2`, `h3`), and
   * assign IPs of `10.0.1.1`, `10.0.2.2`, `10.0.3.3` to the respective hosts.

2. You should now see a Mininet command prompt. Start a ping between h1 and h2:
   ```bash
   mininet> h1 ping h2
   ```
   Because there are no rules on the switches, you should **not** receive any
   replies yet. You should leave the ping running in this shell.
   
3. Open another shell and run the starter code:
   ```bash
   cd ~/tutorials/p4app-exercises/p4runtime.p4app
   make mycontroller
   ```
   This will install the `advanced_tunnel.p4` program on the switches and push the
   tunnel ingress rules.
   The program prints the tunnel ingress and egress counters every 2 seconds.
   You should see the ingress tunnel counter for s1 increasing:
   ```
    s1 ingressTunnelCounter 100: 2 packets
   ```
   The other counters should remain at zero. 

4. Press `Ctrl-C` to the second shell to stop `mycontroller.py`

Each switch is currently mapping traffic into tunnels based on the destination IP
address. Your job is to write the rules that forward the traffic between the switches
based on the tunnel ID.

### Potential Issues

If you see the following error message when running `mycontroller.py`, then
the gRPC server is not running on one or more switches.

```
p4@p4:~/tutorials/p4app-exercises/p4runtime.p4app$ make mycontroller
...
grpc._channel._Rendezvous: <_Rendezvous of RPC that terminated with (StatusCode.UNAVAILABLE, Connect Failed)>
```

You can check to see which of gRPC ports are listening on the machine by running:
```bash
sudo netstat -lpnt
```

The easiest solution is to enter `Ctrl-D` or `exit` in the `mininet>` prompt,
and re-run `make`.

### A note about the control plane

A P4 program defines a packet-processing pipeline, but the rules
within each table are inserted by the control plane. In this case,
`mycontroller.py` implements our control plane, instead of installing static
table entries like we have in the previous exercises.

**Important:** A P4 program also defines the interface between the
switch pipeline and control plane. This interface is defined in the
`advanced_tunnel.p4info` file. The table entries that you build in `mycontroller.py`
refer to specific tables, keys, and actions by name, and we use a P4Info helper
to convert the names into the IDs that are required for P4Runtime. Any changes
in the P4 program that add or rename tables, keys, or actions will need to be
reflected in your table entries.

## Step 2: Implement Tunnel Forwarding

The `mycontroller.py` file is a basic controller plane that does the following:
1. Establishes a gRPC connection to the switches for the P4Runtime service.
2. Pushes the P4 program to each switch.
3. Writes tunnel ingress and tunnel egress rules for two tunnels between h1 and h2.
4. Reads tunnel ingress and egress counters every 2 seconds.

It also contains comments marked with `TODO` which indicate the functionality
that you need to implement.

Your job will be to write the tunnel transit rule in the `writeTunnelRules` function
that will match on tunnel ID and forward packets to the next hop.

![topology](./topo.png)

In this exercise, you will be interacting with some of the classes and methods in
the `p4runtime_lib` directory (absolute directory `~/tutorials/p4app-exercises/p4app/docker/scripts/p4runtime_lib`, or `../p4app/docker/scripts/p4runtime_lib` relative to this exercise's directory). Here is a summary of each of the files in the directory:
- `helper.py`
  - Contains the `P4InfoHelper` class which is used to parse the `p4info` files.
  - Provides translation methods from entity name to and from ID number.
  - Builds P4 program-dependent sections of P4Runtime table entries.
- `switch.py`
  - Contains the `SwitchConnection` class which grabs the gRPC client stub, and
    establishes connections to the switches.
  - Provides helper methods that construct the P4Runtime protocol buffer messages
    and makes the P4Runtime gRPC service calls.
- `bmv2.py`
  - Contains `Bmv2SwitchConnection` which extends `SwitchConnections` and provides
    the BMv2-specific device payload to load the P4 program.
- `convert.py`
  - Provides convenience methods to encode and decode from friendly strings and
    numbers to the byte strings required for the protocol buffer messages.
  - Used by `helper.py`


## Step 3: Run your solution

Follow the instructions from Step 1. If your Mininet network is still running,
you will just need to run the following in your second shell:
```bash
make mycontroller
```

You should start to see ICMP replies in your Mininet prompt, and you should start to
see the values for all counters start to increment.

### Extra Credit and Food for Thought 

You might notice that the rules that are printed by `mycontroller.py` contain the entity
IDs rather than the table names. You can use the P4Info helper to translate these IDs
into entry names. 

Also, you may want to think about the following:
- What assumptions about the topology are baked into your implementation? How would you
need to change it for a more realistic network?

- Why are the byte counters different between the ingress and egress counters?

- What is the TTL in the ICMP replies? Why is it the value that it is?
Hint: The default TTL is 64 for packets sent by the hosts.

If you are interested, you can find the protocol buffer and gRPC definitions here:
- [P4Runtime](https://github.com/p4lang/p4runtime/blob/master/proto/p4/v1/p4runtime.proto)
- [P4Info](https://github.com/p4lang/p4runtime/blob/master/proto/p4/config/v1/p4info.proto)

#### Cleaning up Mininet

Typing `exit` at the `mininet>` prompt should cause the Mininet
process, which was started inside of a new docker container, to exit
and for that container to be removed.

The log files written to the directory `/tmp/p4app-logs` remain after
the container exits, but will be overwritten with new contents when
you next do `make`.

#### Running the reference solution

To run the reference solution, you should run the following command from the
`~/tutorials/p4app-exercises/p4runtime.p4app` directory:
```bash
make solutioncontroller
```
