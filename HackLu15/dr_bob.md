## Dr.Bob  -  150 (+80) Points
by lama (Forensic)

> There are elections at the moment for the representative of the students and the winner will be announced tomorrow by the head of elections Dr. Bob. The local schoolyard gang is gambling on the winner and you could really use that extra cash. Luckily, you are able to hack into the mainframe of the school and get a copy of the virtual machine that is used by Dr. Bob to store the results. The desired information is in the file /home/bob/flag.txt, easy as that.


#### Understanding the given files
1. Zip file with folders home > dr_bob
2. Inside dr_bob there is a hidden folder called VirtualBox
3. This folder contains a VM image, including Virtual Disk Images(.vdi), the machine saved state(.sav) and an XML text file defining the virtual machine(.vbox).

#### Let's get to /home/bob
1. We start by running the VM without it's saved state. Upon booting, the system asks us for the decryption key of the home folder partition;
2. We start the VM, loading its saved state and check that it only asks for user login (Therefore the disk may be already deciphered at that point! Is the encryption key hanging in memory? [Cold Boot Attack comes to mind](https://www.usenix.org/legacy/event/sec08/tech/full_papers/halderman/halderman.pdf));
3. We attach the virtual disk images to another VM under our control and inspect /etc/shadow and /etc/passwd to check user accounts on the machine. As expected, bob is listed as a user. Changing /etc/shadow file on disk seems to have no effect whatsoever on VM reboot.

### Understanding the encryption applied on the partition

* The VM disk attached to our Kali VM is mapped into /dev/vg/home
* Upon some search, we stumble across LUKS-Linux Unified Key Setup
* We run `cryptsetup luksDump /dev/vg/home` and got information about the LUKS device header

![Image of LUKS device](https://octodex.github.com/images/yaktocat.png)

* LUKS has 8 slots to hold keys for a particular device. We state that there is only a key setup on the device, probably pertaining to Bob. The device's masterkey is ciphered with a keyfile/passphrase.
* The masterkey is deciphered and loaded into memory when a user enters his passphrase to unlock the device. This ensures that a legitimate user can access the disk contents without the need to re-insert his passphrase at every file access.
* At this point, we are able to add a key for ourselves if we get to know Bob's passphrase or the device's masterkey. We may also overwrite the device's LUKS headers with knowledge of the masterkey but it is more error-prone and not necessary to complete the challenge.

### Extracting the masterkey from memory
* As before, the VM is started with its saved state loaded. It is possible to dump the VM RAM content by executing `command `.
* Taking advantage of AES keys predictable structure, we use [Aeskeyfind] () to find keys in the memory dump: `./aeskeyfind -v memdump`
* We found a 128 AES key (matching the AES masterkey size see on LUKS header dump) - 1fab015c1e3df9eac8728f65d3d16646
* The key must now be converted to binary format. We did using xxd and stored it into our masterkey file- `echo "1fab015c1e3df9eac8728f65d3d16646" | xxd -r -p > mkf.key`
* Now we just need to use the masterkey as "authenticator" to add our own device key to an unused Key Slot. File AddKEY.txt just contains some random mumble-jumble. We are just interested in choosing the new passphrase. `cryptsetup luksAddKey /dev/vg/home AddKEY.txt --master-key-file mkf.key`
* We now have our own keyfile in place and are able to decipher the partition with our chosen passphrase.
