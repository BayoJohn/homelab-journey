# Day 1 – Building the Foundation of My Homelab

Today marked the beginning of my homelab project. My main objective was not to start installing Docker, Kubernetes, or any other advanced technology. Instead, I wanted to build a solid foundation that could support everything I plan to deploy later.

It can be tempting to begin a homelab project by immediately installing applications and services. However, I decided to take a different approach. I wanted to first understand the environment, configure the network properly, and confirm that the machines could communicate reliably. Once the foundation was stable, I could then begin adding more complex tools without constantly running into avoidable infrastructure problems.

## Setting Up the Hardware

My homelab currently consists of two Dell OptiPlex 3050 systems, each with 16 GB of RAM.

The first system runs Ubuntu Desktop and serves as my management workstation. This is the computer I use to access the lab, connect to servers through SSH, write configuration files, and manage the rest of the environment. As the project grows, I also plan to use this machine for tools such as Terraform, Ansible, Git, and Visual Studio Code.

The second system runs Proxmox Virtual Environment. It acts as the hypervisor and will host the virtual machines used throughout the project.

For now, both computers are connected directly using an Ethernet cable. The Proxmox server does not yet have its own direct internet connection, so the Ubuntu workstation temporarily performs two roles. It acts as my management machine and also serves as the internet gateway for the lab.

The Ubuntu computer receives internet access through USB tethering from my mobile phone. It then shares that connection with the Proxmox network through the Ethernet interface.

This is not the final network design I intend to use, but it gives me a functional starting point. It also gives me the opportunity to work directly with Linux networking concepts such as routing, IP forwarding, firewall rules, and Network Address Translation.

## Creating My First Virtual Machine

After confirming that Proxmox was running, I created my first Ubuntu Server virtual machine.

This virtual machine would serve as the first proper server in the lab and eventually become the base for future systems. I assigned it a static IP address so that it would always remain reachable at the same location on the network.

Using a static IP address is important in a server environment. Services such as SSH, Kubernetes, monitoring tools, and configuration-management systems depend on predictable network addresses. I did not want the server receiving a different IP address every time it restarted.

Once the machine had been created and configured, I attempted to connect to it from my Ubuntu workstation using SSH.

That was when the troubleshooting began.

## The First Problem: SSH Connection Refused

When I tried to connect to the virtual machine, SSH returned a `Connection refused` error.

At first, this looked like a network problem, but the error actually provided a useful clue. A connection refusal usually means the destination machine can be reached, but there is no service listening on the requested port.

In this case, the workstation was successfully reaching the virtual machine, but the SSH server was either not installed or not running.

The obvious next step was to install the OpenSSH server package. However, before I could do that, I needed to update the Ubuntu package repositories.

Running `apt update` revealed another problem.

## Discovering the Internet Connectivity Issue

The package update process was extremely slow and eventually failed. Instead of assuming something was wrong with Ubuntu or the package manager, I began testing the network one layer at a time.

I first checked whether the virtual machine could communicate with other devices on the local network. It could reach both the Proxmox host and the Ubuntu workstation, which confirmed that the internal network configuration was working.

I then tested access to external IP addresses. Those tests failed.

This showed that the problem was not with communication inside the homelab. The virtual machine could send traffic to the Ubuntu workstation, but that traffic was not successfully leaving the workstation through the USB-tethered internet connection.

The Ubuntu workstation was already forwarding packets, but it was not translating the private IP addresses used inside the homelab before sending them to the internet.

When I inspected the firewall configuration, I noticed that Docker had already created several NAT rules for its own bridge networks. However, there was no matching rule for the homelab network, which uses the `10.0.0.0/24` address range.

That missing rule was the main cause of the problem.

## Fixing Routing and NAT

To solve the issue, I created an `iptables` masquerade rule for the homelab network.

The purpose of this rule was to replace the private source IP address of packets leaving the lab with the IP address assigned to the USB-tethering interface. This allowed return traffic from the internet to find its way back through the Ubuntu workstation and into the virtual machine.

I also added forwarding rules to allow traffic to move between the internal Ethernet interface and the external USB interface.

Once the rules were applied, I tested the connection again. This time, the virtual machine could reach external IP addresses. Domain-name resolution also began working, confirming that both internet access and DNS were functioning correctly.

At that point, it appeared that the network problem had been solved.

However, `apt update` still failed.

## The Unexpected System-Time Problem

The new error message stated that the Ubuntu repository release files were “not valid yet.”

This was confusing at first because the virtual machine now had working internet access and DNS. After checking the system date, I discovered that the virtual machine's clock was several months behind the actual date.

Because the server believed it was still an earlier date, the repository metadata appeared to come from the future. Ubuntu therefore rejected it as invalid.

This was an important lesson for me because the problem had nothing to do with networking, even though it appeared immediately after the networking issue. It showed how package management, security verification, certificates, and system time are all connected.

A server with an incorrect clock can experience problems with software repositories, HTTPS connections, authentication systems, logs, and other time-sensitive services.

## What I Achieved

By the end of the session, I had successfully established the basic network infrastructure for the homelab.

The Ubuntu Server virtual machine had a static IP address and could communicate with the Proxmox host and the Ubuntu management workstation. It could also access the internet and resolve domain names through DNS.

Although the system-time issue still needed to be corrected before I could complete the package installation, the most difficult part of the day had been resolved.

The biggest lesson from today was the importance of troubleshooting problems systematically. Instead of changing multiple settings randomly, I tested each part of the network separately.

I confirmed local communication first, followed by external connectivity, NAT, DNS, and finally system time. This made it easier to understand exactly where each problem was occurring and why.

## Next Steps

My next task will be to correct the time synchronization inside the virtual machine. Once that is working, I will install the OpenSSH server and the QEMU Guest Agent.

After SSH is available, I will configure passwordless authentication from the Ubuntu workstation. The final goal is to prepare this virtual machine as a reusable template that can be cloned whenever I need to deploy additional servers in the homelab.

Day 1 was mainly about building the foundation, and although I encountered several problems, each one helped me better understand how the different parts of the environment work together.
