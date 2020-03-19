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

3. Host configuration
    ```bash
    # Change hostname
    sudo hostnamectl set-hostname <some-name>
 
    # Preserve hostname across reboots
    set `preserve_hostname: true` in `/etc/cloud/cloud.cfg`
    ```

4. VM Optimizations - [ref](https://www.kernel.org/doc/Documentation/sysctl/vm.txt)

5. Systemd - Service & Timers - [ref](https://www.freedesktop.org/software/systemd/man/index.html)

6. SSH tunnels and port forwarding -
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

7. Login/Non-Login-Interactive/Non-Interactive shells
    * https://unix.stackexchange.com/questions/38175/difference-between-login-shell-and-non-login-shell 
    * http://mywiki.wooledge.org/DotFiles
    * https://docstore.mik.ua/orelly/unix3/upt/ch03_04.htm
    * https://askubuntu.com/questions/463462/sequence-of-scripts-sourced-upon-login
    
8. Find & Replace - `grep -rl matchstring somedir/ | xargs sed -i 's/matchstring/newstring/g'`

9. List all connected client's IPs, group by count - `netstat -tn 2>/dev/null | grep :80 | awk '{print $5}' | cut -d: -f1 | sort | uniq -c | sort -nr | head`
