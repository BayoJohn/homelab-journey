## Day 1 – Building the Foundation of the Homelab

Today's objective was to establish the foundation upon which the rest of the homelab will be built. Rather than rushing to install technologies such as Docker or Kubernetes, the focus was on creating a stable infrastructure that can reliably host those technologies later. A common mistake made by beginners is to install applications without first understanding the environment in which they operate. The approach taken here was the opposite: build the infrastructure first, verify that it functions correctly, and only then begin deploying services.

### Hardware and Network Architecture

The homelab consists of two Dell OptiPlex 3050 systems, each equipped with 16 GB of RAM:

* **Management Workstation:** Runs Ubuntu Desktop. This machine is responsible for administering the entire lab, accessing servers through SSH, writing Infrastructure as Code, and eventually running tools such as Terraform, Ansible, and Visual Studio Code.
* **Hypervisor:** Runs Proxmox Virtual Environment, a Type-1 hypervisor that hosts all virtual machines required throughout the project.

At this stage, the Ubuntu workstation and the Proxmox server are connected directly using an Ethernet cable. Since the Proxmox server does not currently have direct internet access, the Ubuntu workstation temporarily acts as both the management workstation and the internet gateway. Internet connectivity is provided through USB tethering from a mobile phone connected to the Ubuntu workstation.

Although this architecture is not intended as the final design, it provides a functional environment while allowing valuable exposure to Linux networking concepts such as routing, forwarding, and network address translation (NAT).

### Deploying the First Virtual Machine

Once Proxmox was operational, an Ubuntu Server virtual machine was created to serve as the first server in the environment. This virtual machine was assigned a static IP address to ensure that its network identity remains consistent across reboots.

Static addressing is particularly important in server environments because services such as SSH, Kubernetes, and configuration management tools rely on predictable IP addresses. Unlike home computers, servers rarely obtain new addresses through DHCP every time they boot.

---

### Challenges and Troubleshooting

**1. SSH Connection Refused**
The first challenge arose when attempting to connect to the virtual machine using SSH from the Ubuntu workstation. Instead of establishing a connection, SSH returned a "Connection Refused" error. This message was significant because it indicated that the workstation could successfully reach the virtual machine over the network, but the SSH service itself was unavailable. The error suggested that the OpenSSH server package had either not been installed or was not running. Before installing OpenSSH, however, another problem became apparent.

**2. Network Routing and NAT**
Attempting to update the Ubuntu package repositories using `apt update` proved to be extremely slow and eventually failed. Rather than assuming that Ubuntu itself was broken, a structured troubleshooting process was followed:

* Network connectivity was tested first by attempting to reach known IP addresses on the local network.
* Next, external addresses on the internet were tested.

Further investigation revealed that while communication within the local network functioned correctly, the virtual machine could not reach external networks. The virtual machine correctly forwarded traffic to the Ubuntu workstation, but the Ubuntu workstation was not translating those packets before forwarding them to the USB tether interface. Inspection of the firewall configuration confirmed this suspicion: existing NAT rules had been created automatically by Docker for its own bridge networks, but there was no equivalent rule for the homelab network (10.0.0.0/24).

**The Fix:** The problem was resolved by creating a masquerade rule using `iptables`. This rule instructed the Linux kernel to replace the private source address of packets originating from the homelab network with the address assigned to the USB tether interface before forwarding them to the internet. Additional forwarding rules were also added to allow traffic to move between the internal Ethernet interface and the external USB interface.

*Note: Once these rules were applied and NAT was functioning correctly, hostname resolution and DNS began working as expected.*

**3. System Time Synchronization**
Although internet connectivity and DNS were now fully operational, one final issue prevented successful package installation. Ubuntu reported that the repository release files were "not valid yet." This error was traced to an incorrect system clock inside the virtual machine. The virtual machine's date lagged behind the current date by several months, causing Ubuntu to reject repository metadata that appeared to originate from the future. This illustrated how seemingly unrelated subsystems—such as package management, cryptographic verification, and system time—are closely interconnected in modern operating systems.

---

### Wrap-up and Next Steps

By the end of the session, the networking infrastructure of the homelab had been successfully established. The virtual machine possessed a functional static IP configuration, could communicate with both the Proxmox host and the Ubuntu workstation, had full internet connectivity, successfully resolved domain names, and was ready for the installation of OpenSSH once the system clock issue was corrected.

More importantly, the day's work reinforced the importance of systematic troubleshooting. Rather than applying random fixes, each layer of the network stack was tested independently until the precise cause of failure was identified and corrected.

**Next Stage Objectives:**

* Correct time synchronization.
* Install OpenSSH Server and the QEMU Guest Agent.
* Enable passwordless SSH authentication.
* Prepare the Ubuntu virtual machine to become the reusable template from which future infrastructure components will be deployed.
