---
layout: default
title: "Break the raid"
---

The process of breaking some proxmox servers out of their raid configuration

<!--more-->

## The goal

Get the raid large pools we created for proxmox on the hosts mss1 and mss2 broken out without breaking everything. 

## Why?

I would like to start using ceph inside the cluster and raid arrays don't mix with ceph. I think setting up ceph will help as we scale up the cluster to make our storage more widely available. Also, one of our servers is setup with its operating system on a raid-0 which is not great.

## OK, so how?

We will start with mss2. things we will need to do include...

- pull a dump of the config for the proxmox machine
- checkout [this-proxmox-wiki-article](https://pve.proxmox.com/wiki/Proxmox_Cluster_File_System_(pmxcfs)#_recovery)
  ![test]({{ site.url }}{{ site.baseurl }}/images/Screenshot_20241031_123227_Chrome.jpg)
- migrate all virtual machines off of mss2 and onto mss1.
- remove all data from anything that currently is in ceph
- shutdown the OSB that lives on mss2 and remove it from ceph. ceph will complain about this. that's OK. we will be re-adding stuff before too long.
- shut mss2 down. ensure you can still get to the idrac
- destroy the large raid in the virtual disk sub menu !!! YOU NEED TO MAKE SURE TO DESTROY THE CORRECT ONE OR THIS WILL BE AN ABSOLUTE PAIN !!!
- Do not switch the raid controller out of raid mode. the dell site about the raid controllers were using says it will pass the drives though in this mode while still controlling the raid for the operating system. I guess we'll see if that is in fact true.
- After the raid is broken up, go to the physical disks tab and select non-RAID for the disks you want to pass though to the host system. per the Dell documents on the raid controller were using this should pass the drives straight into the system.
- now you get to see what happened :D
- boot up the host operating system and see what disks are there
- if the new drives show up in proxmox as target-able drives, add them as OSB's
- assuming all this works the migration will be conducted in a similar fashion on mss1

## OK Smart guy, whats the backup plan?

A back out plan should not be necessary so long as no vm's get lost and no data is lost. in the case the disks that used to be in the bulk raid array don't show up, Ill likely have to get back into the raid card to sort it out. It is highly unlikely that proxmox gets wiped off the mss2 system since we will not be touching the raid that holds the disks with proxmox on them. the worst case here is that the drives will be passed though ONLY with the RAID card being setup in HBA mode. that would be a major bummer. we'll probably want to decide if its worth the performance hit to go software raid for the OS and ceph for the rest of the drives. I have a feeling this cluster will never get big enough to justify that trade-off.

## extra info

We have a PERC H730 integrated Raid controllers in both dells here are the documents for them

On Mss2 we have a total of 8 drives with drive 0 and 1 setup in a raid 1 for the boot operating system. the remaining drives 2 - 7 are all 1TB and can be added to the ceph pool, they are currently in a raid 5

On mss1 we have a total of 8 drives with drive 0 and 1 setup in a raid 0 (this is not ideal) for the operating system. drives 2-7 are in a raid 5 and all have 1 TB of capacity

Crash only has 2 drives available. one for the operating system and one spare that's currently in use by ceph. this is fine. It may be a good idea to balance the drives in the cluster a bit. we can do that later

mss1
![test]({{ site.url }}{{ site.baseurl }}/images/Screenshot_20241031_135634_Chrome.jpg)

mss2
![test]({{ site.url }}{{ site.baseurl }}/images/Screenshot_20241031_135627_Chrome.jpg)

## So how did it go with mss2?

Well it ended up going well! A few things we learned...

- when you turn off a Dell system using the ACPI shutdown (not like ripping the power out) the raid card and drives will be off also and you will not be able to manipulate them via the idrac interface.
- next we powered up the server and ended up kicking the OSB on mss2 off the Ceph cluster, and then ensuring that it was not mounted on the proxmox host. Once that was done we went back to the idrac and we're able to delete the raided virtual disk. That fixed the 6 disks from the raid back to physical disks.
- next we were able to see the drives to non raid. We had to do this one by one with about 3-5 minutes in-between since there was some sort of job queue lock. As we added those drives they showed up on the proxmox system. This was all done while the proxmox servers was live.
- once all drives were added individually, we were able to add them to the Ceph cluster as OSB's
- the one thing that kinda sucked there was we again had to add drives one by one. It is probable that if we add the ochestrator Ceph plugin we could add a bunch of drives at the same time but I'm not totally sure. I think I'm going to install that plugin and see about that for the next time
- we determined that we will likely need to re-install mss1 since the operating system currently lives on a raid 0

## Alright so how do we do the thing with mss1?

- Reading online, it seems like you can convert a raid 0 to a raid 1 if you have the spare drives already avalible. What I can do to make this is is in this [link](https://www.dell.com/community/en/conversations/poweredge-hddscsiraid/migrate-raid-0-to-raid-1-how/647f5019f4ccf8a8de8fb5c7)
![test]({{ site.url }}{{ site.baseurl }}/images/Screenshot_20241104_114028_Chrome.jpg)

1. Migrate all Virtual machines off of mss1 (this was more dificult than strictly nessicary since a HA task was setup for zugzug. I turned it off (probably) by removing its uspace group temporarily)
2. remove the OSB from the ceph pool that lives on MSS1
3. Migrate/duplicate the metadata Managment service from mss1 to mss2
4. Ensure the migration tasks have succeeded
5. Check that you have serial access to he server via the idrac ( do have this! Its super cool I'll write more later)
6. shut down mss1
7. install usb thumb drive for a live ubuntu desktop iso image. this is just a place holder so we can have all drives unmounted while the system is turned on. I may be able to do this with a virtual ISO which would be rad as hell
8. Turn on the system using the idrac and boot into the thumb drive. Or virtual drive
9. Access console via vnc to ensure your access and that all drives are disconnected.
10. Once verified, get into the idrac again.
11. Remove the raid 6 virtual disk, this will take a good couple minutes.
12. Next you select the virtual disk (currently the raid 0 disks) and select reconfigure
13. You will be prompted to add drives that are available to create a raid 1 and the data shoullllld transfer over
14. The transfer will take a few hours.
15. Check with your running live mounted system that disks have been copied over and reconfigured correctly.
16. After that Ill need to take all drives that are unused and set them up as non raid disks. Will need to check with Garth if he cares about the boot disks being in specific slots.
17. Once all reconfig tasks are done reboot the system and boot into the raid 1 virtual disk to confirm proxmox functionality.
18. If that functions, the raid 0 virtual disk can be formated, wiped and unmounted from the proxmox os and you can break the raid in the idrac console. Once they are in non-raid mode, go ahead and add them to the Ceph pools

- To get the serial console working I had to install the openManage Mobile app and the bVNC free application. Once those are installed, connect up to the idrac IP address and once connected select the virtual console. This will only work on mss1 since it has the enterprise level licence

- The one additional thing we need to be sure of is that there is a big ass wind storm kicking around Washington right now. That could make power less of a reliable factor. If power goes out during this operation were pretty much maxed out on the UPS so hopefully that wont happen? Gonna chat with Garth about that.

---

So we paused the migration due to the wind storm that was happening on 11/4 and I went ahead with the migration on mss1 here's how that went...

- I had the machine still shut down from yesterday
- I used my Ubuntu with a GUI VM on mss2 to install the icedtea application to run the .jnlp files, interestingly while this worked on the VM in the micro-space, it did not work from my laptop. IDK if there are some firewall/port rules getting hit but something is preventing the virtual console working remotely. Not sure what's going on there but there's that. ( It did work from my phone so who knows. You cant load a virtual disk that way unfortunately though)
- I ended up starting with trying to load parted magic as a virtual disk before that became clear that I wouldn't be able to do that.
- turns out the idrac wont let you configure the raid levels (as far as I can tell) so I needed to get into the bios
- This was done simply enough by setting the "next boot" option to the system bios and rebooting the server. This got me to the bios where I was able to eventually get to a bios configuration wizard (setup utility) and then go into the raid controller and then Virtual disks. This menu allowed me to reconfigure the raid0 array to a raid 1 array. There's a problem with this though
- By adding drives to a raid 0 array to make it a raid 1 array, We have created a raid 10. This is NOT what we want. This is gonna mean that we have 4 whole disks taken up by the boot drive which kinda sucks. So how down we fix this?
- Well I think we have enough drives to create an entirely separate raid 1 array that will be our target.
- Then we will boot into a live system with all the raid arrays unmounted.
- we will have to resize the file system that proxmox lives so that it will fit on a 1  terabyte raid 1 array instead of the raid 10 array of 2 TB. Luckily only about 300 GB is used on the raid 10 array so this shouldn't be a problem. 
- We will use the DD command to copy over all the bytes from the raid 10 to the raid 1 array. This should mean that the raid 1 array becomes a boot-able proxmox volume
- once we verify that proxmox boots off the new raid 1 array Ill delete the raid 10 and add those drives into the Ceph pool
- If none of that works I'll re-install all this stuff. That would kinda suck but that's how it be sometimes.

---

## So how did it go with mss1?

Not as well as I would have liked. There were two things I misunderstood. the first was how simply DD-ing the raid array from one drive to the other wouldn't work well. Maybe there is a way I could have completed this but it would have involved messing with the grub configuration as well as additional changes. It is likely a worthwhile thing to try doing this on a test system in the future but at the time I was unable to complete things without wiping out all the drives.
the other issue was that when converting a raid 0 array to a raid 1 array using the dell idrac tools and the raid controller, there's no way to change the array cleanly from one to the other. what it does instead is convert the whole array to a raid 10 which is a lot of drives and there isn't enough of a performance bump for us to justify this. What I ended up doing was deleting all drives and starting fresh.
The nice thing here was that no critical data was lost. All virtual machines had been moved off of mss1 prior to starting the switch over and re-installing mss1 was a very simple task. all I really had to do was to make sure to set the IP address correct, set the root password the same as what we had previously and then re-add mss1 into the proxmox cluster.
There were some things I had to do to ceph to get it working correctly again, basically I uninstalled everything and started fresh with ceph. Now, we have 12 terabytes of ceph storage available accost the cluster. Crash really probably should have some drives moved onto it and we probably need to get some faster networking on that system to make the whole cluster act a bit better. load balancing small ceph clusters is critical and ours is somewhat poorly optimized at the moment.

In all I learned a whole bunch about getting access to the dell servers IDRAC interfaces, how to shuffle data around and I learned a bit about coming up with reasonable back-out plans. This whole episode really showed how  excellent proxmox's clustering tools work.
