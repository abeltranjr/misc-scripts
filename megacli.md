#### Install
https://chiranjeevigk.wordpress.com/2017/08/01/install-lsi-megariad-storage-manger-on-proxmox/

1: Install MegaRaid CLI

Add repo

edit /etc/apt/sources.list add the entry
```
deb http://hwraid.le-vert.net/debian jessie main
```
here i am installing on debian (promox in debian based) and jessie version (proxmox 4.4.1)

Then save the file and run the below command to trust or sign the above site.
```
wget -O - https://hwraid.le-vert.net/debian/hwraid.le-vert.net.gpg.key | apt-key add -
```
run the command
```
apt-get install megacli
```
#### MegaCLI: useful commands

Recently I installed a server with a Supermicro SMC2108 RAID adapter, which is actually a LSI MegaRAID SAS 9260. LSI created a command line utility called MegaCLI for Linux to manage this adapter. You can download it from their support pages. The downloaded archive contains an RPM file. I installed mc and rpm on Debian with apt-get, and then extracted the MegaCli64 binary (for x86_64) to /usr/local/sbin, and the libsysfs.so.2.0.2 from the Lib_utils RPM to /opt/lsi/3rdpartylibs/x86_64/ (that’s the location where MegaCli64 looks for this library).

Here are some useful commands:
View information about the RAID adapter

For checking the firmware version, battery back-up unit presence, installed cache memory and the capabilities of the adapter:
```
# MegaCli64 -AdpAllInfo -aAll
```
View information about the battery backup-up unit state

```
# MegaCli64 -AdpBbuCmd -aAll
```
View information about virtual disks

Useful for checking RAID level, stripe size, cache policy and RAID state:

```
# MegaCli64 -LDInfo -Lall -aALL
```
View information about physical drives

```
# MegaCli64 -PDList -aALL
```
Patrol read

Patrol read is a feature which tries to discover disk error before it is too late and data is lost. By default it is done automatically (with a delay of 168 hours between different patrol reads) and will take up to 30% of IO resources.

To see information about the patrol read state and the delay between patrol read runs:
```
# MegaCli64 -AdpPR -Info -aALL
```
To find out the current patrol read rate, execute
```
# MegaCli64 -AdpGetProp PatrolReadRate -aALL
```
To reduce patrol read resource usage to 2% in order to minimize the performance impact:
```
# MegaCli64 -AdpSetProp PatrolReadRate 2 -aALL
```
To disable automatic patrol read:
```
# MegaCli64 -AdpPR -Dsbl -aALL
```
To start a manual patrol read scan:
```
# MegaCli64 -AdpPR -Start -aALL
```
To stop a patrol read scan:
```
# MegaCli64 -AdpPR -Stop -aALL
```
You could use the above commands to run patrol read in off-peak times.
Migrate from one RAID level to another

In this example, I migrate the virtual disk 0 from RAID level 6 to RAID 5, so that the disk space of one additional disk becomes available. The second command is used to make Linux detect the new size of the RAID disk.
```
# /usr/local/sbin/MegaCli64 -LDRecon -Start -r5 -L0 -a0
# echo 1 > /sys/block/sda/device/rescan
```
Extending an existing RAID array with a new disk
```
./MegaCli64 -LDRecon -Start -r5 -Add -PhysDrv[32:3] -L0 -a0
```
Create a new RAID 5 virtual disk from a set of new hard drives

First we need to now the enclosure and slot number of the hard drives we want to use for the new RAID disk. You can find them out by the first command. Then I add a virtual disk using RAID level 5, followed by the list of drives I want to use, specified by enclosure:slot syntax.
```
# MegaCli64 -PDList -aALL | egrep 'Adapter|Enclosure|Slot|Inquiry'
# MegaCli64 -CfgLdAdd -r5'[252:5,252:6,252:7]' -a0
```
Extending an existing RAID array with a new disk

First check the enclosure device ID and the slot number of the newly added disk with the command above. Then we reconstruct the logical drive, adding the new drive. For a RAID 5 array this command is used:
```
# MegaCli64 -LDRecon -Start -r5 -Add -PhysDrv[32:3] -L0 -a0
```
View reconstruction progress

When reconstructing a RAID array, you can check its progress with this command.
```
# MegaCli64 -LDRecon ShowProg L0 -a0
```
(replace L0 by L1 for the second virtual disk, and so on)
Configure write-cache to be disabled when battery is broken
```
# MegaCli64 -LDSetProp NoCachedBadBBU -LALL -aALL
```
Change physical disk cache policy

If your system is not connected to a UPS, you should disable the physical disk cache in order to prevent data loss.
```
# MegaCli -LDGetProp -DskCache -LAll -aALL
```
To enable it (only do this if you have a UPS and redundant power supplies):
```
# MegaCli -LDGetProp -DskCache -LAll -aALL
```
More information


################################################################################3
Adapter parameter -aN
The parameter -aN (where N is a number starting with zero or the string ALL) specifies the PERC5/i adapter ID. If you have only one controller it’s safe to use ALL instead of a specific ID, but you’re encouraged to use the ID for everything that makes changes to your RAID configuration.

Physical drive parameter -PhysDrv [E:S]
For commands that operate on one or more pysical drives, the -PhysDrv [E:S] parameter is used, where E is the enclosure device ID in which the drive resides and S the slot number (starting with zero). You can get the enclosure device ID using “MegaCli -EncInfo -aALL”. The E:S syntax is also used for specifying the physical drives when creating a new RAID virtual drive.

Virtual drive parameter -Lx
The parameter -Lx is used for specifying the virtual drive (where x is a number starting with zero or the string all).

Controller information

    MegaCli -AdpAllInfo -aALL
    MegaCli -CfgDsply -aALL
    MegaCli -AdpEventLog -GetEvents -f events.log -aALL && cat events.log

Enclosure information

    MegaCli -EncInfo -aALL 

Virtual drive information

    MegaCli -LDInfo -Lall -aALL 

Physical drive information

    MegaCli -PDList -aALL
    MegaCli -PDInfo -PhysDrv [E:S] -aALL 

Battery backup information

    MegaCli -AdpBbuCmd -aALL 

Controller management

Silence active alarm

    MegaCli -AdpSetProp AlarmSilence -aALL 

Disable alarm

    MegaCli -AdpSetProp AlarmDsbl -aALL 

Enable alarm

    MegaCli -AdpSetProp AlarmEnbl -aALL 

Physical drive management

Set state to offline

    MegaCli -PDOffline -PhysDrv [E:S] -aN 

Set state to online

    MegaCli -PDOnline -PhysDrv [E:S] -aN 

Mark as missing

    MegaCli -PDMarkMissing -PhysDrv [E:S] -aN 

Prepare for removal

    MegaCli -PdPrpRmv -PhysDrv [E:S] -aN 

Replace missing drive

    MegaCli -PdReplaceMissing -PhysDrv [E:S] -ArrayN -rowN -aN 

The number N of the array parameter is the Span Reference you get using “MegaCli -CfgDsply -aALL” and the number N of the row parameter is the Physical Disk in that span or array starting with zero (it’s not the physical disk’s slot!).

Rebuild drive

    MegaCli -PDRbld -Start -PhysDrv [E:S] -aN
    MegaCli -PDRbld -Stop -PhysDrv [E:S] -aN
    MegaCli -PDRbld -ShowProg -PhysDrv [E:S] -aN

Clear drive

    MegaCli -PDClear -Start -PhysDrv [E:S] -aN
    MegaCli -PDClear -Stop -PhysDrv [E:S] -aN
    MegaCli -PDClear -ShowProg -PhysDrv [E:S] -aN

Bad to good (or back to good as I like to call it)

    MegaCli -PDMakeGood -PhysDrv[E:S] -aN 

This changes drive in state Unconfigured-Bad to Unconfigured-Good.

Walkthrough: Change/replace a drive

Set the drive offline, if it is not already offline due to an error

    MegaCli -PDOffline -PhysDrv [E:S] -aN 

Mark the drive as missing

    MegaCli -PDMarkMissing -PhysDrv [E:S] -aN 

Prepare drive for removal

    MegaCli -PDPrpRmv -PhysDrv [E:S] -aN 

Change/replace the drive

If you’re using hot spares then the replaced drive should become your new hot spare drive:

    MegaCli -PDHSP -Set -PhysDrv [E:S] -aN

In case you’re not working with hot spares, you must re-add the new drive to your RAID virtual drive and start the rebuilding

    MegaCli -PdReplaceMissing -PhysDrv [E:S] -ArrayN -rowN -aN
    MegaCli -PDRbld -Start -PhysDrv [E:S] -aN
    
#################################################################
#################################################################


Replacing an LSI raid disk with MegaCli

If you have identified a failed, or failing disk, it is possible to replace it using the MegaCli utility. In the example below we will cover replacing a failed disk from a raid 5 that has three disks total.

The first thing we want to check is the status of our raid 5.

[root@raid log]# MegaCli64 -ldinfo -lALL -aALL
 Adapter 0 — Virtual Drive Information:
 Virtual Drive: 0 (Target Id: 0)
 Name :
 RAID Level : Primary-5, Secondary-0, RAID Level Qualifier-3
 Size : 929.458 GB
 Parity Size : 464.729 GB
 State : Degraded
 Strip Size : 64 KB
 Number Of Drives : 3
 Span Depth : 1
 Default Cache Policy: WriteBack, ReadAheadNone, Cached, No Write Cache if Bad BBU
 Current Cache Policy: WriteThrough, ReadAheadNone, Cached, No Write Cache if Bad BBU
 Default Access Policy: Read/Write
 Current Access Policy: Read/Write
 Disk Cache Policy : Disk’s Default
 Encryption Type : None
 Is VD Cached: Yes
 Cache Cade Type : Read Only

You can see in the example above that the state of the array is showing up as ‘State : Degraded’. This means that at least one disk has failed, or is not present in the array. Next we will want to look at all of our disks:

[root@raid log]# MegaCli64 -pdlist -aALL

The output of that command is quite long, but in our example it shows three disks and their primary information is:

Enclosure Device ID: 252
 Slot Number: 0
 ….
 Firmware state: Online, Spun Up

Enclosure Device ID: 252
 Slot Number: 1
 ….
 Firmware state: Online, Spun Up

Enclosure Device ID: 252
 Slot Number: 2
 ….
 Firmware state: Online, Spun Up

Enclosure Device ID: 252
 Slot Number: 3
 ….
 Firmware state: Offline

In our example the failed disk is shown as ‘Enclosure Device ID:252′ and ‘Slot Number: 3′. So for MegaCli syntax this drive will be reference as [252:3] in the examples below. Now that we know the EIDs and slot numbers of each of the drives we can go ahead and remove the failed drive.

1) First we set the original disk offline if an error has not already cause the controller to set it offline

[root@raid log]# MegaCli64 -pdoffline -physdrv[252:3] -a0
Adapter: 0: EnclId-252 SlotId-3 state changed to OffLine.
Exit Code: 0x00

2) Mark the failed disk as missing

[root@raid log]# MegaCli64 -pdmarkmissing -physdrv[252:3] -aAll

EnclId-252 SlotId-3 is marked Missing.

Exit Code: 0x00

3) Mark the failed disk as prepared for removal

[root@raid log]# MegaCli64 -pdprprmv -physdrv[252:3] -a0
Prepare for removal Success
Exit Code: 0x00

4) Now you can go replace the faulty disk, it might help to use the hdd identify command to locate the disk

[root@raid log]# MegaCli64 -pdlocate -start -physdrv[252:3] -a0

Adapter: 0: Device at EnclId-252 SlotId-3 — PD Locate Start Command was successfully sent to Firmware
Exit Code: 0x00

Step 5 has two options below

5a) If you use hot spares and the original hot spare was already put into the raid array, set the new disk to replace the hot spare that just went into service

# MegaCli64 -PDHSP -Set -PhysDrv[<enclosure#>:<disk#>] -a<adapter#>

5b) If you don’t use hot spares you will need to add the disk to the array and start the rebuild manually

# MegaCli64 -PdReplaceMissing -PhysDrv[252:3] -Array0 -row0 -a0
# MegaCli64 -PDRbld -Start -PhysDrv[252:3] -a0

6) Optional: We can watch the rebuild progress. Depending on the size of the array this may take a considerable amount of time. Also the raid array is usable during this time, but you can expect to encounter performance hits while the raid array is rebuilding.

# MegaCli64 -PDRbld -ShowProg -PhysDrv[252:3] -a0

####################################################################################################
####################################################################################################

https://hwraid.le-vert.net/wiki/DebianPackages
http://ftzdomino.blogspot.com/2009/03/some-useful-megacli-commands.html
https://twiki.cern.ch/twiki/bin/view/FIOgroup/DiskRefPerc
http://hwraid.le-vert.net/wiki/LSIMegaRAIDSAS
http://kb.lsi.com/KnowledgebaseArticle16516.aspx
http://erikimh.com/megacli-cheatsheet/
http://www.advancedclustering.com/act_kb/replacing-a-disk-with-megacli/
-- Replace disk
https://www.ibm.com/support/knowledgecenter/en/ST5Q4U_1.6.2/com.ibm.storwize.v7000.unified.162.doc/spi_t_pfa_2.html
