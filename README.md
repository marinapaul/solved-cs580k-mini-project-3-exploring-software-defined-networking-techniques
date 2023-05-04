Download Link: https://assignmentchef.com/product/solved-cs580k-mini-project-3-exploring-software-defined-networking-techniques
<br>
<h1>1       Summary</h1>

In this assignment, we will play with software-defined (SDN) networking techniques and get familiar with the main concepts of SDN. We will use Mininet to create an emulation platform with the software-based OpenFlow switches (i.e., Open vSwitch). On top of this emulated SDN platform, POX is used as the SDN controller. We need to correctly deploy the emulation platform and SDN controller, be familiar with the operations, program SDN controllers, and finish the required tasks. After these tasks, we should have a concrete idea on how SDN controller communicates with OpenFlow switches.

<h1>2       Get to Know the Tools</h1>

2.1      Experimental Environment

In this assignment, we will the Ubuntu Linux VM provided by Walton School, or any other Linux machine but please make sure the Ubuntu version is 18.04 LTS image (that way, the TAs can help you with the configuration if you need help).

Note: please switch to the “root” user for this assignment, as we notice that mininet container can only work properly under the root user.

2.2     Mininet Installation

Mininet [1] is a great way to develop, share, and experiment with OpenFlow and Software-Defined Networking systems. To not mess our native network (e.g., your GCP instance), we will run all experiments using a docker instance.

We can pull Mininet docker and start it simply by:

$ docker pull iwaseyusuke/mininet

$ docker run -it –name=mininet –privileged iwaseyusuke/mininet

We may need to practice and get familiar with Mininet by following [2] (skip the Wireshark part). Particularly, we need to understand how to define a custom topology using Python scripts (e.g., “custom/topo-2sw2host.py” in [2]). After walking through the main operations of Mininet, please finish Task I.

2.3      Task I: Define a Custom Topology

We know that data center networks typically have a tree-like topology. End-hosts connect to top-of-rack switches, which form the leaves of the tree; one or more core switches form the root; and one or more layers of aggregation switches form the middle of the tree. For example, a binary tree network of depth 3 looks like in Figure 1. Task I asks you to develop a Python script that can populate such a binary tree topology in the Mininet. After competing the Python script, you should run the following commands to verify it — “h1” should reach “h5” with ping.

$ mn –custom binary_tree.py –topo mytopo

$ mininet&gt; h1 ping h5

2.4       SDN Controller (POX) Installation

POX [4] is an open source development platform for Python-based software-defined networking (SDN) control applications. In this assignment, we use POX as the OpenFlow controller <a href="#_ftn1" name="_ftnref1"><sup>[1]</sup></a>.

Open a new terminal of the running Mininet docker:

$ docker exec -it mininet /bin/bash

<ol>

 <li>Download POX</li>

</ol>

$ apt-get update

$ apt-get install git

$ env GIT_SSL_NO_VERIFY=true git clone https://github.com/noxrepo/pox

$ apt-get install python3

<ol>

 <li>Start the SDN controller</li>

</ol>

$ cd pox

$ python3 pox.py log.level –DEBUG misc.of_tutorial

This loads the controller in ∼/pox/pox/misc/of_tutorial.py. This controller acts like a “dumb” switch and floods all packets.

<ol>

 <li>Connect our network topology to the controller</li>

</ol>

At the same time, create our binary-tree topology (completed in Task I) that connects to the controller using the following command (hint: you need to execute another bash for the same mininet container):

$ mn –custom binary_tree.py –controller remote –topo mytopo

This connects all software switches (s1 ∼ s6) to the controller. Please take a look at the output from the controller console.

2.5       Task II: Study the “of_tutorial” Controller

We have learned that, for the SDN setup, if there are no forwarding rules for a particular packet, such a packet will be sent to the SDN controller (i.e., “of_tutorial” controller). Please understand this concept by studying the “of_tutorial” controller, and answer the following questions.

<ul>

 <li>1 Draw the function call graph of this controller. For example, once a packet comes to the controller, which function is the first to be called, which one is the second, and so forth?</li>

 <li>2 Have h1 ping h2, and h1 ping h5 for 100 times (e.g., h1 ping -c100 p2). How long does it take (on average) to ping for each case? What is the difference, and why?</li>

 <li>3 Run “iperf h1 h2” and “iperf h1 h5”. What is “iperf” used for? What is the throughput for each case? What is the difference, and why?</li>

 <li>4 Which of the switches observe traffic (s1 ∼ s6)? Please describe the way for observing such traffic on switches (e.g., adding some “print” functions in the “of_tutorial” controller).</li>

</ul>

2.6    Resources

Notic that this document only provides a barebone guide to the setup. You are encouraged to look through the following documents:

https://github.com/noxrepo/pox http://mininet.org/walkthrough/ https://openflow.stanford.edu/display/ONL/POX+Wiki http://therandomsecurityguy.com/openvswitch-cheat-sheet/ https://github.com/mininet/openflow-tutorial/wiki

<h1>3       Task III: MAC learning controller</h1>

In Task II, we have learned a controller that makes switches flood every packet they receive, and thus, in practice, behave like hubs. In this section, we will change this behavior such that the switches will learn the ports packets arrive from, and upon receiving a packet, if they have already seen its destination address, they will know the exact port to forward it on and avoid flooding the network (i.e., MAC learning controller).

Please add the following code into “of_tutorial” controller (type the code down manually). This realizes the above MAC learning controller. Please change other function(s) accordingly to make sure “act_like_switch” function will be invoked in your code, analyze the behaviors, and answer the questions.

def act_like_switch (self, packet, packet_in):

# Learn the port for the source MAC

# print “Src: “,str(packet.src),”:”, packet_in.in_port,”Dst:”, str(packet.dst) if packet.src not in self.mac_to_port:

print (“Learning that ” + str(packet.src) + ” is attached at port ” + str(packet_in.in_port)) self.mac_to_port[packet.src] = packet_in.in_port

# if the port associated with the destination MAC of the packet is known:

if packet.dst in self.mac_to_port: # Send packet out the associated port print (str(packet.dst) + ” destination known. only send message to it”) self.resend_packet(packet_in, self.mac_to_port[packet.dst])

else:

# Flood the packet out everything but the input port # This part looks familiar, right?

print (str(packet.dst) + ” not known, resend to everybody”) self.resend_packet(packet_in, of.OFPP_ALL)

<ul>

 <li>1 Please describe how the above code works, such as how the “MAC to Port” map is established. You could use a ‘ping’ example to describe the establishment process (e.g., h1 ping h2).</li>

 <li>2 (Please disable your output functions, i.e., print, before doing this experiment) Have h1 ping h2, and h1 ping h5 for 100 times (e.g., h1 ping -c100 p2). How long did it take (on average) to ping for each case? Any difference from Task II (the hub case)?</li>

 <li>3 Run “iperf h1 h2” and “iperf h1 h5”. What is the throughput for each case? What is the difference from Task II?</li>

</ul>

<h1>4       Task IV: MAC learning controller with OpenFlow rules</h1>

We will try to make the switch a bit smarter. Modify the controller from Section 3 so that when it learns a mapping, it installs a flow rule on the switch to handle future packets. Thus, the only packets that should arrive at the controller are those that the switch doesn’t have a flow entry for. Please realize such a MAC learning controller. The useful piece of code (i.e., how to insert a flow rule to the OpenFlow switch) is shown as follows.

#load the flow module fm = of.ofp_flow_mod()

#fill the match filed (we match the flow with its destination address) fm.match.dl_dst = packet.dst

# Add an action to send to the specified port # You need to figure the out_port by yourself action = of.ofp_action_output(port=out_port) fm.actions.append(action) # Send message to switch self.connection.send(fm)

<ul>

 <li>1 Have h1 ping h2, and h1 ping h5 for 100 times (e.g., h1 ping -c100 p2). How long does it take (on average) to ping for each case? Any difference from Task III (the MAC case without inserting flow rules)?</li>

 <li>2 Run “iperf h1 h2” and “iperf h1 h5”. What is the throughput for each case? What is the difference from Task III?</li>

 <li>3 Please explain the above results — why the results become better or worse?</li>

 <li>4 Run pingall to verify connectivity and dump the output.</li>

 <li>5 Dump the output of the flow rules using “ovs-ofctl dump-flows” (in your container, not mininet). How many rules are there for each OpenFlow switch, and why? What does each flow entry mean (select one flow entry and explain)?</li>

</ul>

<h1>5       Task V: Layer-3 routing</h1>

The task will allow us to experience a bit of layer-3 routing. This is a simplified version of an IP router. We can use “h1∼h5 ifconfig” to find out that the IP addresses of h1∼h5 are from 10.0.0.1 to 10.0.0.5. Install IP-matching rules on switch 6, but let other switches stay as MAC learning switches (same as Section 3). The goal is that all hosts from h1 to h4 can ping h5, and switch 6 forwards packets via IPmatching flow rules (not MAC-matching rules). You can install IP-matching rules in a controller or using ovs-ofctl commands (hint: figure it out how to use this command).


