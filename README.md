Various SDN Projects (Based on GT-CS6250)
=========
These programs are to be run on the current version of **Mininet** (either running on a VM or through native installation).
Documentation for setting up **Mininet** can be found at: [the Mininet Website](http://mininet.org/download/).
To run any of these programs, first clone this repository into the **Mininet** machine. Also, all of them are written in Python, so please make sure you have Python installed. 

## Project: Parking Lot
This program takes as an argument, N, and builds a software defined network topology with N nodes.
To check this program out, in the ~/Building_Network_Topology_with_N_nodes directory, run: 
```bash
$sudo python parkinglot.py --bw <link_bandwidth> --dir <output_dir> -t <expt_duration> -n <n>
```
where n is the number of nodes to be on the network. To see a graphical results with random n, check out one of my output folders with a timestamp in the directory. 
and it will create test graphs and 
### A brief overview of my implementation
The meat of this program is found at parkinglot.py and the code is pretty straightforward. Mininet has many useful and simple interfaces that allow you to efficiently set up the topology. For example, the Topo package allows you to initiate a network with one single line of code:
```bash
Topo.__init__(self, **params)
```
and it also has various methods such as addSwitch(), addHost(), addLink() that add switches, hosts and links to the existing topology, respectively. Therefore, you can set up a topology with 2 nodes simply by running 4 lines of code:
```bash
curSwitch = self.addSwitch('s1');
curHost = self.addHost('h1',**hconfig)
self.addLink(receiver, curSwitch, port1=downlink, port2=uplink, **lconfig)
self.addLink(curHost, curSwitch, port1=0, port2=hostlink, **lconfig)
```
Now, to make a topology accomodate a variable number of nodes, just add a for loop:
```bash
for i in range(2, n+1):
		switchName = 's'+str(i)
		hostName = 'h'+str(i)
		nextSwitch = self.addSwitch(switchName)
		nextHost = self.addHost(hostName, **hconfig)
		self.addLink(curSwitch, nextSwitch, port1=downlink, port2=uplink, **lconfig)
		self.addLink(nextHost, nextSwitch, port1=0, port2=hostlink, **lconfig)
		print(switchName + ' added')
		print(hostName + ' added')
		curSwitch = nextSwitch
		curHost = nextHost
```
And that's the most important part of the program and everything else is a normal backbone code. All in all, it is pretty straightforward.
## Project: Programmable Firewall
This program takes as an argument a list of MAC addresses and modifies the OpenFlow of the existing SD network and block them programmatically. 
The only problem with this project was that Pyretic is no longer maintained so I had to use an older version of the language which caused compatibility issues. So make sure you have Python-3.0.1 installed for this one to work. 
To test this program out, first make sure you have POX and Pyretic installed (because the program is written in those languages) and then run (taking the POX program as an example):
```bash
$cp firewall-policies.csv pox_firewall.py ~/pox/pox/misc   \\copy the firewall policy
$pox.py forwarding.l2_learning misc.pox_firewall           \\copy the POX code
$sudo mn --topo single,3 --controller remote --mac \\initiale a Mininet network
mininet> h1 ping -c1 h2    \\tests out the firewall by pinging them
mininet> h1 ping -c1 h3
```
This will use pre-existing policy (the .csv file) and programmatically enforce them through the POX program. When the network initializes, h1 (host 1) cannot ping h2 but can ping h3 because h1 is blocked by h2 by the specified policy. 
### A brief overview of my implementation
The main part (the firewall) of the program is at pox_firewall.py:
```bash
def _handle_ConnectionUp (self, event):
  policies = self.read_policies(policyFile)
    for policy in policies.itervalues():
	    block = of.ofp_match()
	    block.dl_src = policy[0]
	    block.dl_dst = policy[1]
	    flow_mod = of.ofp_flow_mod()
	    flow_mod.match = block
	    event.connection.send(flow_mod)
```
For each of the network instances, the POX code iterates through the existing policies and ensure that it satisfies those properties. In this case, it is analyzing the list of policies and using ofp_match() function (provided by POX) and adding them to a "blacklist" in the OpenFlow controller. Then, when the OpenFlow switch, controlled by the OpenFlow controller, encounters a connection that matches the destination and source addresses in the blacklist, that packet is dropped entirely. 
## Project: DoS Mitigator
This program utilizes libraries, sFlow (a framework for analyzing network) and PyResonance (a framework for altering network based on defined events), to detect and respond to a Denial of Service (DoS) attack.
To test this program out, first make you have sFlow and PyResonance installed on your machine (because the program is written utiziling those frameworks) and then run:
```bash
$sudo mn --controller=remote --topo=single,3 \\start the mininet network
$sudo ovs-vsctl -\- --id=@sflow create sflow agent=eth0 target=\"127.0.0.1:6343\" sampling=2 polling=20 -\- -\- set bridge s1 sflow=@sflow \\create a local sFlow agent
$sudo ./start.sh    \\start sFlow
$./pyretic.py pyretic.pyresonance.main --config=./pyretic/pyresonance/global.config --mode=manual   \\start the DoS mitigation app
mininet>pingall       \\there is no DoS attack, so this ping should go through
mininet>xterm h1      \\create an xterm window of h1
$sudo ping 10.0.0.2 -i .05    \\start an attack by causing a Denial of Service
mininet>pingall       \\now, the node h1 should be blocked because it caused DoS
```
### A brief overview of my implementation
In this example, we see that when the Mininet is initialized, sFlow is initiated, and the DoS mitigation app is created, every host can ping each other without interruption. However, when h1 starts pinging h2 with 20 times a second, it gets "marked" by sFlow. The most important part of this program is sFlow. In this case, the sFlow has a built-in event types that detects network changes. For example, a message that is equal to EVENT_TYPES['ddos'] in sFlow specifies that the source is suspected of sending large volume of TCP SYN's and then dropping the connection. If this happens, the sFlow is programmed into putting them in the "ddos-attacker policy". Once we have the list of all attackers, we can simply set up another policy, in dos_policy.py, where we only allow connection without that attacker:
```bash
def block_policy(self):
    return drop
        
def allow_policy(self):
    return passthrough
    
def action(self):
    if self.fsm.trigger.value == 0:
      listofhosts = self.fsm.get_policy("ddos-attacker")
	    passthrough = ~listofhosts	            
      return passthrough
    else:
      return self.turn_off_module(self.fsm.comp.value)
```
## My Experience
Through implementing these projects and taking the class GT-CS6250 in general, I learned a lot about important networking principles ranging from the most basic to the most advanced, with Software Defined Networking being the most valuable one. Working on them has been a pleasure, so thank you Professor Feamster for this wonderful course!
