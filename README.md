# Tailscale on DD-WRT

To set up [Tailscale](https://tailscale.com) on a DD-WRT capable router, you'll need to configure the router to support *direct-attached storage* (DAS) and then install and configure *Tailscale* from Entware. Here's a step-by-step guide to help you through the process.

## Prerequisites

- **DD-WRT firmware installed:** Ensure your router has DD-WRT installed. You can find the firmware and installation instructions on the DD-WRT website.
- **External storage device:** You’ll need a USB drive or an external hard drive to connect to your router.
- **Basic networking knowledge:** Familiarity with DD-WRT and basic networking concepts.

## Step-by-Step Instructions

1. Partition and format an external hard drive
    - The process is out of the scope of this tutorial because there are already a lot of information available on internet.
    - In the following we are using a single EXT4 partition.
      
2. Connect to your router
    - Access your DD-WRT router’s web interface by entering the router's IP address (usually 192.168.1.1) in your web browser.
    - Log in with your admin username and password.

3. Connect and mount the external storage

    - Plug your USB drive or external hard drive into the router’s USB port (USB3 port is better).
    - Go to **Services > USB** in the DD-WRT web interface.
        - Enable *Core USB Support*
        - Enable *USB Storage Support*
        - Enable *Automatic Drive Mount*
        - **Apply Settings** and the disk will be temporary mounted on `/tmp/mnt/sda1`
        - Copy and paste UUID from Disk Info to *Mount partition to /opt*
        - **Save**
        - **Apply Settings**
   -  Go to **Administration > Managment**
        - **Reboot Router** and your drive should be automatically mounted on `/opt`
     
        ![USB](https://github.com/alf45tar/DD-WRT-TimeMachine/blob/main/images/Services-USB.jpg)

4. Connect to your router via Terminal

   On macOS open a Terminal and run:
   ```
   ssh root@192.168.1.1
   ```
   Enter `root` password when requested.
   ```
   alf45tar@alf45tar-iMac ~ % ssh root@192.168.1.1
   DD-WRT v3.0-r56182 std (c) 2024 NewMedia-NET GmbH
   Release: 05/02/24
   Board: TP-Link Archer C9
   root@192.168.1.1's password: 
   ==========================================================
 
         ___  ___     _      _____  ______       ____  ___ 
        / _ \/ _ \___| | /| / / _ \/_  __/ _  __|_  / / _ \
       / // / // /___/ |/ |/ / , _/ / /   | |/ //_ <_/ // /
      /____/____/    |__/|__/_/|_| /_/    |___/____(_)___/ 
                                                     
                           DD-WRT v3.0
                       https://www.dd-wrt.com


    ==========================================================


    BusyBox v1.36.1 (2024-05-02 04:40:12 +07) built-in shell (ash)

    root@TailscaleRouter:~#
    ```
   
5. Intall Entware the ultimate repo for embedded devices
   
   [Entware](https://entware.net) is a software repository for embedded devices like routers or network attached storages. >1800 packages are available. It was founded as an alternative to very outdated Optware packages.
   List of Entware packages for ARMv7 is [here](http://bin.entware.net/armv7sf-k3.2/Packages.html).

   Type in the following commands:
   ```
   cd /opt
   wget http://bin.entware.net/armv7sf-k3.2/installer/generic.sh
   sh generic.sh
   ```
   When installation is complete, run an update:
   ```
   opkg update
   opkg upgrade
   ```
   
6. Install required packages
   ```
   opkg install tailscale_nohf
   ```
   It seems that the default `tailscale` package tries to execute some instructions which are not available on all processors. The `_nohf` variant takes care of this.

7. Edit `/opt/etc/init.d/S06tailscaled` to append `--tun=userspace-networking` to `ARGS`
   ```
   #!/bin/sh

   ENABLED=yes
   PROCS=tailscaled
   ARGS="--state=/opt/var/tailscaled.state --statedir=/opt/var/lib/tailscale --tun=userspace-networking"
   PREARGS="nohup"
   DESC=$PROCS
   PATH=/opt/sbin:/opt/bin:/opt/usr/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
   
   . /opt/etc/init.d/rc.func
   ```
   It appears that the `tun` kernel module is missing and not available on Entware, at least not for DD-WRT. Luckily, it looks like tailscale's `userspace-networking` option still works.
   
8. Run Entware services
    ```
    /opt/etc/init.d/rc.unslung start
    ```

9. Run `tailscale version` command to check `tailscaled` daemon installation
    ```
    root@tailscalerouter:~# tailscale version
    1.78
      tailscale commit: 1b41fded
      go version: go1.23.4
   ```

10. Follow normal `tailscale` node installation command
    ```
    tailscale up
    ```
    or for act as subnet router
    ```
    tailscale up --advertise-routes=192.168.1.0/24
    ```
    
13. Add startup and shutdown scripts

    - Go to **Adminstration > Commands** in the DD-WRT web interface.

        Add the following to **Startup**
        ```
        # Wait until external storage is mounted under /opt
        /bin/sh -c 'until [ -f /opt/etc/init.d/rc.unslung ]; do sleep 1 ; done'
        # Start all Entware services
        /opt/etc/init.d/rc.unslung start
        ```

        Add  the following to **Shutdown**
        ```
        # Stop all Entware services before shutdown
        /opt/etc/init.d/rc.unslung stop
        ```

14. Reboot your Router for the last time
    -  Go to **Administration > Managment**
        - **Reboot Router**

15. After reboot enjoy your tailnet.
