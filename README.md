# Home Network 🏡

The purpose of this repository is to highlight the growth in my home network's capability and sophistication.

What prompted this development was an old HP Omen gaming desktop that was just standing there, menacingly waiting to be utilized for something other than a paperweight.

## Physical Setup

<img width="992" height="710" alt="IMG_5146" src="https://github.com/user-attachments/assets/183c6d16-a455-4743-9efb-79ea32198590" />

Behold, a very ugly homelab setup. The intent was to spend the least amount of money to get the system up and running.

Hardware:
- Old HP Omen Gaming desktop with a fried ethernet port from a lightning strike
- USB-to-Ethernet port adapter
- Spectrum default modem and router
- Netgear GS305E Managed switch
- APC UPS

Future plans:
- Cannibalizing the gaming desktop into a more compact frame where I can insert the computer into a server rack.
- Install PCIe Network Card to replace the fried ethernet port usb adapter solution.
- Purchase Unifi Gateway router
- Buy larger UPS with auto shutdown capabilities

## Line Diagram of Network 

<img width="582" height="278" alt="image" src="https://github.com/user-attachments/assets/f63e36b7-f0e5-4b1d-a549-154203ffc220" />

## Services

- Proxmox does both jobs: it’s a hypervisor for VMs (KVM) and a container host for LXC.
- Samba → LXC container: a simple Linux service, so lightweight is ideal.
- Kali → VM: it does low-level network work and benefits from its own kernel and full isolation.
- Greenbone → Docker containers inside a VM. The Docker containers share the Debian VM’s kernel, not Proxmox’s. That combination gets the convenience of Docker plus the strong isolation of a VM wrapped around it.
