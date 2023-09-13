## Process Monitoring

- [ps](https://www.sysadmin.md/ps-cheatsheet.html) \- Report a snapshot of the current processes.
    
- [top](https://gist.github.com/ericandrewlewis/4983670c508b2f6b181703df43438c37) \- Display Linux processes.
    
- [htop](https://www.maketecheasier.com/power-user-guide-htop/) \- Interactive process viewer.
    
- atop - Advanced interactive monitor to view the load on a Linux system.
    
- [lsof](https://www.golinuxcloud.com/lsof-command-in-linux/) - List open files.
    

## Performance Monitoring

- nmon - A system monitor tool for Linux and AIX systems.
- iostat - A tool that reports CPU statistics and input/output statistics for devices, partitions and network filesystems.
- [sar](https://www.geeksforgeeks.org/sar-command-linux-monitor-system-performance/) - A system monitor command used to report on various system loads, including CPU activity, memory/paging, device load, network.
- [vmstat](https://www.geeksforgeeks.org/vmstat-command-in-linux-with-examples/) - A tool that reports virtual memory statistics.

## Networking Tools

- [traceroute](https://www.geeksforgeeks.org/traceroute-command-in-linux-with-examples/) - Traces the route taken by packets over an IP network.
- [ping](https://www.geeksforgeeks.org/ping-command-in-linux-with-examples/) - Sends echo request packets to a host to test the Internet connection.
- [mtr](https://www.tecmint.com/mtr-a-network-diagnostic-tool-for-linux/) - Combines the functionality of traceroute and ping into a single diagnostic tool.
- [nmap](https://www.freecodecamp.org/news/what-is-nmap-and-how-to-use-it-a-tutorial-for-the-greatest-scanning-tool-of-all-time/) - Scans hosts for open ports.
- [netstat](https://quickref.me/netstat.html) - Displays network connections, routing tables, interface statistics, masquerade connections, and multicast memberships.
![](./images/netstat.png)
- [iptables](https://erravindrapawadia.medium.com/iptables-tutorial-beginners-to-advanced-guide-to-linux-firewall-839e10501759) -  Firewall utility that uses policy chains to allow or block traffic that will be enforced by the linux kernel’s netfilter framework. 
- [tcpdump](https://opensource.com/article/18/10/introduction-tcpdump) - Dumps traffic on a network.
- [dig](https://www.cyberciti.biz/files/pdf/dig%20command%20cheat%20sheet.pdf) - DNS lookup utility.
- [ip](https://linuxopsys.com/wp-content/uploads/2023/07/ip-cheat-sheet.pdf) - Show or manipulate routing, devices, and tunnels.
  
- scp - Secure copy
  ![Alt text](./images/scp.png)
- rsync - fast and efficient way to copy and synchronize data across directories, disks, and networks, and can be used for data backups
  - --dry-run: allows you to perform a trial run without making any changes to the system
  - -a: archive mode, which includes all the necessary options like copying files recursively, preserving almost everything (like symbolic links, file permissions, user & group ownership and timestamps).
  - -v: verbose mode, which provides more detailed output during the transfer process.  
  - -z: compresses the file data during the transfer.
  - -P: tells rsync to keep partially transferred files if the transfer is interrupted, so that it can resume the transfer from where it left off instead of starting over from the beginning and tells rsync to display a progress bar during the transfer, showing the percentage of the transfer completed and the estimated time remaining

  ### [rsync vs scp](https://stackoverflow.com/questions/20244585/how-does-scp-differ-from-rsync)


## Text Manipulation

- [awk](https://linuxize.com/post/awk-command/) - A programming language designed for text processing and typically used as a data extraction and reporting tool.
- [sed](https://www.geeksforgeeks.org/sed-command-in-linux-unix-with-examples/) - A stream editor for filtering and transforming text.
- [grep](https://www.geeksforgeeks.org/grep-command-in-unixlinux/) - A command-line utility for searching plain-text data sets for lines that match a regular expression.
- [sort](https://www.geeksforgeeks.org/sort-command-linuxunix-examples/) - A command-line utility for sorting lines of text files.
- [cut](https://bencane.com/2012/10/22/cheat-sheet-cutting-text-with-cut/) - A command-line utility for cutting sections from each line of files.
- [uniq](https://www.geeksforgeeks.org/uniq-command-in-linux-with-examples/) - A command-line utility for reporting or omitting repeated lines.
- [cat](https://www.tecmint.com/cat-command-linux/) - A command-line utility for concatenating files and printing on the standard output.
- [tr](https://linuxopsys.com/topics/tr-command-in-linux) - A command-line utility for translating or deleting characters.
- nl - A command-line utility for numbering lines of files.
- [wc](https://onecompiler.com/cheatsheets/wc) - A command-line utility for printing newline, word, and byte counts for files.

## Storage Management

- blkid: List of device uuid
- lsblk: List block devices
- [du](https://www.redhat.com/sysadmin/du-command-options): Disk usage information  
- df -hi: Displays information about file system disk space and inode usage.
- ncdu: Disk usage analyzer with an ncurses interface
- fdisk: create, delete, resize, change and move partitions on the hard drive
    - sudo fdisk -l: list the partition table of all available disks on a Linux system
    - sudo fdisk /dev/sdx ->
      - p: print the current partiton table
      - m: use it to find what command you want
- mkfs: format the partition with whatever file system type you want
   - mkfs.exfat -> using on mulitple os
   - mkfs.ext4 -> using on linux machine
- mount/umount: 
    - in mount 

> [!NOTE] 
> /media is supposed to be the mount point for removable media while /mnt is for temporary mounts initiated by the user
  
## Storage Modes

<img src="./images/storage-types.png" style="max-width: 600px">

|  | SAN | NAS | DAS |
|---|---|---|---|
| **Type of storage** | Blocks | Shared files | Sectors |
| **Transmission of data** | Fiber Channel | Ethernet, TCP/IP | IDE/SCSI |
| **Speed** | 5-10 ms | 20-50 ms | 5-10 ms |
| **Complexity** | High | Moderate | Easy |
| **Mode of access** | Servers | Clients or Servers | Clients or Servers |
| **Capacity** | &gt; 10^12 bytes | 10^9-10^12 bytes | 10^9 bytes |
| **Usage** | Application data | Unstructured, Shared data | OS |


## LVM
![Alt text](/images/lvm.png)
* pvdisplay: gives you information about physical volume
* vgdisplay: display details about the volume group
* lvdisplay: logical volume
* pvcreate: convert hard drvie into a physical volume for lvm
* vgextend: if you have free physical volume you can extend volume group with this command 
* lvextend: extend logical volume
* resize2fs: extend filesystem
<nl>
lvextend --resizefs -l +100%FREE /dev/mapper/vg_name ->
used to extend the size of a Logical Volume and resize the filesystem associated with it

## NFS 

* * *
    
##  vim 
    h,j,k,l -> move cursor 
    x,X -> delete one char
    r -> replace one char 
    o,O -> open new line
    a -> append (you can use it instead of i)
    u -> undo
    dw -> delete a word
    dd -> delete a line
    J -> join two lines
    G -> go to line(G, 12G, 1G)
    y -> yank/copy (yy, yw) 
    p, P(paste)
    / -> search - then you can use 'n' to go next match
    $ -> end of the line 
    ^ -> start of the line 
    . -> repeat last thing you done 
    v -> visual mode (you can select) 
    : set nubmer/nonumber -> for add/remove line number

* * *

-  [tmux](https://danielmiessler.com/p/tmux)
    

-  ssh:
    - ssh logs stores in /var/log/auth.log
    - in ~/.ssh/config file you can add your ssh server details to improves the efficiency, security, and manageability of your SSH connections 
    
    - you should add your public key in .ssh/authorized_keys directory on server
    - -i: you can add your pulic key file by this option
    - -p: for setting the ssh port
    - ssh config file on server stores in /etc/ssh/sshd-config
    - ssh-keygen: use this command to create a new private/public key
      - -C: for adding comment into your pulic key
      -  -t dsa | ecdsa | ecdsa-sk | ed25519 | ed25519-sk | rsa Specifies the type of key to create.
    ### ssh port forwarding
  
<br>

- [chmod](https://quickref.me/chmod.html)
- [find](https://www.geeksforgeeks.org/find-command-in-linux-with-examples/)
- watch: Runs the specified command repeatedly and displays the results on standard output
- truncate: shrink or extend the size of a file to the specified size
- diff: compare files line by line

---
## Systemd
unit directories sorted by their priority:
1. /etc/systemd/system
2. /run/systemd/system
3. /lib/systemd/system

for more information [click here](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/8/html/configuring_basic_system_settings/assembly_working-with-systemd-unit-files_configuring-basic-system-settings)
