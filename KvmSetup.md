# Setup KVM for implementing Windows Server 2016 R4 (CUI) as a Virtual Machine

## Abstract
- This guide provides how to setup KVM for implementing Windows Server 2016 R4 as a VM.

- To implement KVM, you must use CPUs on physical machine compatible with VT-x (Intel-VT or AMD-V).

## System Overview
### System Requirement
- 1 physical machine (Linux)
- 2 virtual machines (Windows Server) on KVM
- 1 shared disk (iSCSI target on host OS)

### System Configuration
- Linux OS: CentOS 7.4.1708
- Windows OS: Windows Server 2016 R4


- Compiled against library: libvirt 3.9.0
- Using library: libvirt 3.9.0
- Using API: QEMU 3.9.0
- Running hypervisor: QEMU 1.5.3


- iSCSI target: targetcli version 2.1.fb46
- iSCSI initiator: Microsoft iSCSI Initiator Version 10.0 Build 17134


- EXPRESSCLUSTER X 4.0

```bat
<LAN>
 |                                               +-------------------+
 |                                               |Virtual Machine    |
 |                                  +------------+- Windows Server   |
 |                                  |            |- iSCSI initiator  |
 |                                  |            |- EXPRESSCLUSTER X |
 |                                  |            +-------------------+
 |  +--------------------+          |
 |  |Physical Machine    |     +----+---+
 +--|- CentOS            +-----+ virbr0 |
 |  |- KVM               |     +----+---+
 |  |- iSCSI target      |          |
 |  +--------------------+          |
 |                                  |            +-------------------+
 |                                  |            |Virtual Machine    |
 |                                  +------------+- Windows Server   |
 |                                               |- iSCSI initiator  |
 |                                               |- EXPRESSCLUSTER X |
 |                                               +-------------------+
```

## System setup
### KVM setup
1. Install some KVM softwares

    ```bat
    $ yum -y install libguiestfs libvirt libvirt-client python-virtinst qemu-kvm virt-manager virt-top virt-viewer virt-who virt-install bridge-utils
    ```
    
    - After installation of KVM, virtual NIC "virbr0" is created.
    
    - This is the gateway of a default virtual network.
    
    - Its IP address is 192.168.100.1 (default).
2. Construct a virtual network

    First of all, launch the "virt-manager"

    ```bat 
    $ virt-manager
    ```
    
    - You can construct virtual environments and operate virtual machines with virt-manager.
    
    1. Click "Edit" and then "Connection Details".
    
    2. Click "Virtual Networks" tab.
    
    3. Click "+"(Add Network) button.
    
    4. Type "Network Name" and then define its IP address.
    
        - By Checking "Enable DHCPv4", you can use DHCP.
        
        - In this case, virtual DHCP assigns the IP addresses in the range you select to virtual machines.
    
    5. Click "Forwarding to physical network".
    
        - The gateway of the virtual network is connected to the destination NIC you select.
        
    6. Click "Finish".
    
3. Create virtual machines

    1. Click "File" and then "New Virtual Machine".

    2. "Local install media" and then select a ISO image.
    
    3. Define Memory, CPUs, Storages and the network you created before.
    
4. Edit a configuration of virtual machines

    - You can edit a configuration of virtual machines by clicking "Open" after right-clicking servers.
    
### Windows setup
1. Disable firewall

    ```bat
    > netsh advfirewall set allprofiles state off
    ```
    
    or on PowerShell
    
    ```bat
    > PS C:> Get-NetFirewallProfile | Set-NetFirewallProfile -Enabled false
    ```
    
2. Configure computer name, network settings
	
	- You can configure these settings using the "sconfig" command.
	
	```bat
	> sconfig
	```
	
3. Configure date and time

    ```bat
    > timedate.cpl
    ```
    
4. Configure region and language

    ```bat
    > intl.cpl
    ```

5. Configure disk settings

    ```bat
    > diskpart
    ```

### iSCSI setup

- Target (Linux)

    1. Install iSCSI target configuration tool
        
        ```bat
        $ yum install -y targetcli
        ```
        
    2. Set target.service auto-startup
    
        ```bat
        $ systemctl enable target.service
        ```
        
    3. Create a device used as iSCSI target
    
        - I created LVM logical volume "LogVol00" in LVM volume group "VolGroup01".
        - After this part, I use LVM as iSCSI target.
        
    4. Define a backstore
    
        - Register the LVM as a block device
        
        ```bat
        $ targetcli /backstores/block create name=<block device name> dev=/dev/VolGroup01/LogVol00
        ```
        - Confirm a target configuration
        
        ```bat
        $ targetcli ls
        ```
        
    5. Define IQN (iSCSI Qualified Name)
    
        - Please confirm by yourself how to name IQN properly.
        
        ```bat
        $ targetcli /iscsi create iqn.2018-09.com.iscsi01:target01
        ```
        
    6. Connect IQN with a backstore
    
        ```bat
        $ targetcli /iscsi/iqn.2018-09.com.iscsi01:target01/tpg1/luns create /backstores/block/<block device name>
        ```
        
    7. Define ACL (Access Control List)
    
        ```bat
        $ targetcli /iscsi/iqn.2018-09.com.iscsi01:target01/tpg1/acls create <initiator IQN>
        ```
        
        - How to confirm initiator IQN is shown below (Windows command).
        
        - "iqn.xxx.xxx.xxx:xxx" is initiator IQN.
        
        ```bat
        > iscsicli
        Microsoft iSCSI Initiator Version 10.0 Build 17134
        
        [iqn.xxx.xxx.xxx:xxx] Enter command or ^C to exit
        ```
        
    8. Define IP address
    
        ```bat
        $ targetcli /iscsi/iqn.2018-09.com.iscsi01:target01/tpg1/portals create <IP address> 3260
        ```
        

- Initiator (Windows)

    1. Set msiscsi auto-startup
    
        - The space after "=" is necessary.
        
        ```bat
        > sc config msiscsi start= auto
        > sc start msiscsi
        ```
        
    2. Add iSCSI target
    
        ```bat
        > iscsicli AddTargetPortal <IP address> <port(default 3260)>
        ```
        
    3. Display target list
    
        ```bat
        > iscsicli ListTargets
        ```
        
    4. Login target
    
        ```bat
        > iscsicli QLoginTarget <IQN>
        ```
        
    5. Set persistent connection
    
        ```bat
        > iscsicli PersistentLoginTarget <IQN> T * * * * * * * * * * * * * * * 0
        ```
        
    6. Display persistent connection
    
        ```bat
        > iscsicli ListPersistentTargets
        ```
        
### Shared disk setup

- Configure disk partitions using "diskpart"
    
    ```bat
    > diskpart
    ```
    
- Display disk list
    
    ```bat
    DISKPART> list disk
    ```
    
- Create partition

    ```bat
    DISKPART> select disk <disk number>
    DISKPART> online disk
    DISKPART> create partition primary size=<size(MB)>
    DISKPART> format fs=ntfs quick
    DISKPART> assign letter=<drive letter>
    ```

### Windows Server command

- Disable NIC

    ```bat
    PS C:> Disable-NetAdapter -Name "<NIC name>"
    ```
    
- Enable NIC

    ```bat
    PS C:> Enable-NetAdapter -Name "<NIC name>"
    ```
    
- Open multiple command-prompts

    - Ctrl+Alt+Delete
    
    - Select "Task Manager"
    
    - Select "Run new task" after right clicking on "Windows Command Processor"
    
    - Type "cmd" and then click "OK"

### References

- Windows setup

    - http://uxg10.clusterpro.nes.nec.co.jp:83/clusterpro/index.php?ServerCore#c7951188
    

- iSCSI
    
    - http://ossfan.net/setup/linux-28.html
    
    - http://yoshifumi.hateblo.jp/entry/20080930/p1
    
    - https://www.upken.jp/kb/hyper-v-server-with-iscsi.html