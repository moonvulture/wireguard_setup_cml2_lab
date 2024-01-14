# WireGuard VPN Setup Guide for EC2 Instances

This guide provides steps to set up a WireGuard VPN on an Amazon EC2 instance for secure connectivity to a home network via a Fritz!Box router.

## Quick Steps Overview

1. Generate `wg0.conf` file from Fritz!Box.
2. Create an EC2 security group with WireGuard ports.
3. Copy `wg0.conf` file to `/etc/wireguard/wg0.conf` on the EC2 instance.
4. Activate the VPN: `wg-quick up wg0`.
5. Add public SSH keys to `authorized_keys` and configure the SSH config file.
6. Add support for deprecated encryption algorithms.

## Detailed Steps

### Setting Up WireGuard VPN

#### For Home Network Connection:

1. **Fritz!Box Configuration:**
   - Log into the Fritz!Box with a user account configured with an authenticator app (e.g., Google Authenticator).
   - Navigate to `Internet -> Permit Access -> WireGuard`.
   - Click `Add New` then select `Single Connection`.
   - Upon completion, download the configuration file. Note the port number under `Peer -> Endpoint`.

      Example Configuration:
      ```
      [Interface]
      PrivateKey = <omitted>
      Address = 192.168.178.156/24
      DNS = 192.168.178.1
      DNS = fritz.box
      Table = off  # Add this line

      [Peer]
      PublicKey = <omitted>
      PresharedKey = <omitted>
      AllowedIPs = 192.168.178.0/24,0.0.0.0/0
      Endpoint = fritze_generated_dns.myfritz.net:53171  # Note this port number
      PersistentKeepalive = 25
      ```

   - Add `"Table = off"` to the interface section to disable iptables rules (necessary for distributions like Amazon Linux 2023).

2. **AWS EC2 Configuration:**
   - In the AWS EC2 console, create a new security group with the following rules:
     - Inbound: TCP 22 (SSH) from Anywhere IPv4, Custom UDP with the port number from the Fritz!Box configuration.
     - Outbound: Allow all (Any port to 0.0.0.0/0).
   - Launch an EC2 instance with the new security group, e.g., using Amazon Linux 2023.

3. **WireGuard Installation and Configuration on EC2:**
   - Connect to the EC2 instance.
   - Install wireguard-tools: `sudo dnf install -y wireguard-tools python`.
   - As root, create and edit `/etc/wireguard/wg0.conf`, pasting the content from the Fritz!Box configuration.
   - Exit the root shell.
   - Activate the VPN: `sudo wg-quick up wg0`.
   - Verify connectivity by pinging the Fritz!Box gateway (usually `192.168.178.1`).

### SSH Key Configuration for Visual Studio Code (VSC)

1. **Generate SSH Keys:**
   - Execute `ssh-keygen` in the terminal or PowerShell.
   - Set permissions: `chmod 600 ~/.ssh/id_rsa`.

2. **Configure SSH Client:**
   - Edit or create `~/.ssh/config` with your EC2 instance details:
     ```
     Host ec2_pyats
       IdentityFile ~/.ssh/id_rsa
       HostName <Your EC2 Instance IP>
       User ec2-user
       PasswordAuthentication no
     ```
   - Replace `<Your EC2 Instance IP>` with your actual EC2 IP address and adjust the `Host` name as desired.

3. **Authorize SSH Keys on EC2:**
   - Open `id_rsa.pub` and copy its contents.
   - On the EC2 instance, create or edit `~/.ssh/authorized_keys` and paste the contents.
   - Save and exit. You should now be able to connect using the pubkey.

### Enabling Deprecated Encryption Algorithms

**Context:**
- Modern versions of Linux, including the one used in this guide, do not support Diffie-Hellman Group 1. This lack of support can cause SSH connection failures to older iOS and IOSv devices.

**Solution:**
- To re-enable deprecated encryption algorithms on your EC2 server, modify one of the following files:
  - `/etc/ssh/ssh_conf`: Add Diffie-Hellman Group 1 to the key exchange.
  - `~/.ssh/config`: Add individual device configurations as needed.
