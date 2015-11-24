# EgressChecker

EgressChecker is a mini-framework which can be used to help check for egress filtering between your host and a client system on which you have the ability to execute commands and scripts.

## Summary
Most penetration testers have, at one time or another, had the need to measure or circumvent data egress security controls; red team exercises, targeted attack simulations and some internal penetration tests are good examples. For the purposes of simplicity, assume that there is an opportunity to execute code on either a Windows or a UNIX machine; the next stage would be to effectively 'portscan' your machine's IP address from the compromised machine in order to find out which ports can be used to egress data. 

This is not a new problem; there are a large number of scripts and tools designed to meet this need and the majority do this very well. This tool aims to improve on the principles on which the other tools have been built, offering a few additional features in a slightly more user-friendly way. 

I wanted a tool that could offer something for both windows and UNIX systems, that would present the commands and tools to run as one-liners, could offer both TCP and UDP, that could take advantage of native tools on the client, that could format the results and does not require me to immediately kill all my existing listeners.  
 
## Components

This tool comprises two main components:
* Generating packets on all ports in a given range, which would be run on the compromised host (the client). For example, assume that the client's address is 10.0.0.1.
* Measuring which connections were made to your machine (the attacker). For example, assume that your IP address is 192.168.0.1.

## Mechanism

On the basis of the example above, this tool will allow you to connect to 192.168.0.1 on each port from 10.0.0.1, and let you see which connections were successful, which effectively will identify gaps or breaches in firewalls.

The basic approach is to:

* Generate a 'one-liner' that can be run on the client. Currently, EgressChecker can only generate one-liners in python, but I'll add other scripts in the fullness of time.
* Use tcpdump to monitor connections to your machine. EgressChecker will print the command that you need to run to perform the necessary capturing and filtering. If used in TCP mode, it just looks for SYN packets. Tcpdump will be configured to save the filtered capture file.
* Parse the tcpdump file, from which the results can be displayed in a number of formats, useful for other tools or simply for reporting. Currently, *tshark* is used as a pcap parser; EgressChecker provides the parameters to pass to tshark.

### Client script

Regardless of the scripting language used, code that is generated by this framework will generate packets according to the following routine:

* The metadata (target IP address, port range, number of threads etc to use) will be 'hardcoded' into the script. The script is semi-dynamically generated based on the parameters provided.
* If THREADS is set to 1, this effectively means that threading will be disabled and the code that manages threads will not be included. 
* If THREADS is greater than 1, the script will import a threading library if necessary, divide up the ports to be scanned between the threads and launch all of them.
 * For example, if all ports between 10 and 20 were included and 4 threads were configured, the script would assign the 4 threads to scan the following ports:
   * Thread 1: 10, 14, 18
    * Thread 2: 11, 15, 19
    * Thread 3: 12, 16, 20
    * Thread 4: 13, 17
 * Each thread will loop through the requested ports and, depending on the configuration, will attempt to connect() (for TCP) and sendto() (for UDP). If both protocols are requested, the script will attempt the TCP connection first, followed by the UDP scan. 
   * In the example above, Thread 1 will connect to TCP/10, UDP/10, TCP/14, UDP/14, TCP/18, UDP/18 and then exit.
  * The main script will exit once all threads have completed and will block until then.

## FAQs
### Why tcpdump?

There are a number of different approaches that others have implemented and that I've tried. However, I have had mixed results from them. 

One tool used scapy to sniff packets; despite being a very elegant approach, I found that it did not seem to capture all packets. 

Another tool worked by opening up listening sockets on a large number of ports on your machine, and monitoring connections to them. Nothing wrong with it, but you can't do too many at once, and I wanted a solution that would sniff the packets (meaning that you may not have to drop your software firewall, nor would you need to kill any existing listeners that you have).

A different tool worked by using iptables to redirect all traffic to a single port, and monitored that port. I couldn't use it because FreeBSD doesn't have iptables, so it did not work for me.

Using tcpdump has a few other benefits too; for example, it does not require listeners to be set up. In addition, this technique could easily be used on machines several layers deep on a target network. If access to a machine has been compromised through a pivot with sufficient privileges to dump traffic, internal egress could be assessed too.

### Is it stealthy?

Not really. You can configure a delay between packets being generated if you want to. I've tried to keep the client code (one liners) as small as possible; they really are very simple scripts.
