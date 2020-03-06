# Linux Utilities
<img src="https://github.com/abhishektripathi24/platform-setup/blob/master/linux/images/linux-logo.png" width="430" height="200"/>

To know more about linux, visit https://www.linux.org/

## Overview
From the official docs -

> Linux is a family of open source Unix-like operating systems based on the Linux kernel, an operating system kernel first released on September 17, 1991, by Linus Torvalds. Linux is typically packaged in a Linux distribution.

## Administration
1. User Management -
    * Sudo user with password-less login
        ```bash
        adduser <username>
        echo '<username> ALL=(ALL) NOPASSWD:ALL' > /etc/sudoers.d/<username>
        sudo su <username>
        vim .ssh/authorized_keys > '<public key of username>'
        id # verify user and group id after creation
        ```
    * Delete user
         ```bash
        sudo deluser -f --remove-home <username>
        ```

2. Mount external volumes - [ref](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ebs-using-volumes.html)
    * Manual process to be carried out once -
        ```bash
        # To view your available disk devices and their mount points. The output of lsblk removes the `/dev/` prefix from full device paths.
        lsblk
    
        # Get information about a device, such as its file system type
        sudo file -s /dev/xvdf
    
        # Create a new file system if not present
        sudo yum install xfsprogs # (optional if XFS tools are already present)
        sudo mkfs -t xfs /dev/xvdf 
    
        # Create data dir and mount volume
        sudo mkdir /data
        sudo mount /dev/xvdf /data
        ```
    * Automatically mount volume after os reboot -
        ```bash
        # Get the UUID of the device.
        sudo blkid

        # Append the following like in /etc/fstab
        sudo vim /etc/fstab
        UUID=<UUID of the device>  /data  xfs  defaults,nofail  0  2

        # Test
        sudo umount /data
        sudo mount -a
        df -h
        ```

3. SSH tunnels and port forwarding -
    ```bash
    # Local SSH Port Forwarding
    ssh -v -N -i .ssh/riv_devops.pem -L 8080:10.8.77.122:9090 ec2-user@10.8.77.122
    ssh -v -N -i .ssh/riv_devops.pem -L localhost:8080:10.8.77.122:9090 ec2-user@10.8.77.122
    
    # Remote SSH Port Forwarding
    ssh -v -N -i .ssh/riv_devops.pem -R 8080:localhost:8090 ec2-user@10.8.77.122 
    ssh -v -N -i .ssh/riv_devops.pem -R 10.8.77.122:8080:localhost:8090 ec2-user@10.8.77.122
    NOTE: set `GatewayPorts yes` in `/etc/ssh/sshd_config` to open the ports on 0.0.0.0 instead to 127.0.0.1
    
    # Dynamic SSH Port Forwarding (Socks Proxy)
    ssh -i .ssh/riv_devops.pem -D 1080 -C -q -N -v ec2-user@10.8.77.122
    ```
    