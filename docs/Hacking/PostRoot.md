# Post-Root Procedures

!!! warning "Stop!"
    Do not follow any post-root procedure unless explicitly told to in a previous step of this guide.

## Bank Planning

We are now going to prepare an *optimal* bank plan for the same firmware version you have now booted.

Run the following command to look at your Gateway's bank state:

```find /proc/banktable -type f -print -exec cat {} ';' -exec echo ';'```

You should see something like this being printed:

```bash
...
/proc/banktable/booted
<take note of this>
/proc/banktable/active
<take note of this>
...
```

If the above command failed, your board is not dual bank and no bank planning is possible. Otherwise, take note of the current `active` and `booted` banks.
This guide will let you setup your Gateway to boot the same firmware you are now running as per *optimal* bank plan. An optimal bank plan looks like this:

```bash
/proc/banktable/active
bank_1
/proc/banktable/activeversion
Unknown
/proc/banktable/booted
bank_2
/proc/banktable/bootedversion
xx.x.xxxx-...
```

We will now do what your current state requires to match the above one, such that the device will boot from the recommended bank on every reboot.

!!! question "Which bank should I _use_ to stay safe?"
    It's strongly recommended to stick to the above mentioned *optimal* bank plan before applying any further mod to your device. The bigger picture description is available [here](https://github.com/Ansuel/tch-nginx-gui/issues/514). The short thing is that you should really mod your preferred firmware version (not necessarily of *Type 2*) only while it's being booted from `bank_2` and keep `bank_1` as the active one.
    **Key Point**: it's unsafe to deeply mod any firmware which is being booted from `bank_1`.

!!! danger "Notable exception: Missing RBI"
    In the unfortunate case that there are no RBI firmware files available for your board, you are not in a safe position because you can't rely on `BOOTP` firmware recovery. The *optimal* bank plan relies on `BOOTP` to flash and clean-boot a firmware from `bank_1`. Your best option is to ensure you keep a copy of a rootable *Type 2* firmware on both banks and to avoid modding one of them. Please, check the firmware repository page for any available RBI and stop following this bank planning guide if there are none.
    In case some RBI firmwares are available but none of them is of *Type 2*, the hereby described *optimal* bank plan is still fine, despite some limitations.

Run the following commands:

```bash
# Ensure two banks match in sizes
[ $(grep -c bank_ /proc/mtd) = 2 ] && \
[ "$(grep bank_1 /proc/mtd | cut -d' ' -f2)" = \
"$(grep bank_2 /proc/mtd | cut -d' ' -f2)" ] && {
# Clone and verify firmware into bank_2 if applicable
[ "$(cat /proc/banktable/booted)" = "bank_1" ] && {
mtd -e bank_2 write /dev/$(grep bank_1 /proc/mtd | cut -d: -f1) bank_2 && \
mtd verify /dev/$(grep bank_1 /proc/mtd | cut -d: -f1) bank_2 || \
{ echo Clone verification failed, retry; exit; } }
# Make a temp copy of overlay for booted firmware
cp -rf /overlay/$(cat /proc/banktable/booted) /tmp/bank_overlay_backup
# Clean up jffs2 space by removing existing old overlays
rm -rf /overlay/*
# Use the previously made temp copy as overlay for bank_2
cp -rf /tmp/bank_overlay_backup /overlay/bank_2
# Activate bank_1
echo bank_1 > /proc/banktable/active
# Make sure above changes get written to flash
sync
# Erase firmware in bank_1
mtd erase bank_1;
# Emulate system crash to hard reboot
echo c > /proc/sysrq-trigger; }
# end
```

If everything went good, the Gateway will intentionally crash. Wait for it to reboot completely.
You should now be in the previously mentioned *optimal* bank plan. On each reboot, your device will try booting `active` bank first. Since we set `bank_1` as active and we also erased `bank_1` firmware, it will boot from `bank_2`.

!!! hint "Upgrade now!"
    Would you like to upgrade to a newer firmware? This is the perfect moment for doing it. It is now safe to also install *non-Type 2* firmwares. Just follow the [Safe Firmware Upgrade](../../Upgrade/) guide for this. You could also upgrade later in future and continue tweaking the current firmware. Once you did install the updated firmware, come back here and continue reading.

At this point, you now need to check if your [SSH server setup](#setting-up-permanent-ssh-server) is permanent.

## Setting up Permanent SSH Server

Are you connected to SSH on port `6666`? In most cases, the answer is "No, I didn't need to specify port 6666, I've got connected on default port 22". You don't need to run the below commands if you can already connect on default port `22`, your SSH setup is permanent already.

If the answer is "Yes", run these commands to setup a permanent SSH access on port `22` by defining a new dropbear instance:

```bash
uci -q delete dropbear.afg
uci add dropbear dropbear
uci rename dropbear.@dropbear[-1]=afg
uci set dropbear.afg.enable='1'
uci set dropbear.afg.Interface='lan'
uci set dropbear.afg.Port='22'
uci set dropbear.afg.IdleTimeout='600'
uci set dropbear.afg.PasswordAuth='on'
uci set dropbear.afg.RootPasswordAuth='on'
uci set dropbear.afg.RootLogin='1'
uci commit dropbear
/etc/init.d/dropbear enable
/etc/init.d/dropbear restart
```

Now proceed to changing your [root password](#change-the-root-password), this is **mandatory**.

## Change the Root Password

!!! warning "Serious hint!"
    Do not ignore this step! Your firmware was probably designed to work only on certain specific ISP network. Some kind of remote SSH access could be left open by design in such a way only that same ISP could access. Connecting to some different ISP network could lead to this open access to be exposed on the internet.

Run:

```bash
passwd
```

Now you **must** harden your access, to prevent it from being lost because of limited RBI availability or automatic firmware upgrades in future. See [Hardening Root Access](../../Hardening/) page.



## Optional Device Backup

While backing up your device's filesystem is optional, it's also recommended in case something goes wrong.

The following instructions require a USB flash drive (with about 1.5GB free space - device dependent). For alternative or more detailed instructions please refer to [Recourses page](../../Recourses/#making-dumps).


### Setting the Devices Date
The first step in this process is optional, but might be handy in future to determine when your backups were make. If your device has lost (or never recieved) the current time than all the timestamps of any created file will be incorrect. Setting the time with the following command will ensure the correct timestamps are applied to your backup files.

```bash
date -s=2022.04.01-23:01:00
```

The above command sets the date/time to April 1st, 2022 at 11:01pm. To set the correct date adjust the date string accordingly.

### Checking the USB Path
The _script_ that will perform the backup assumes the USB flash drive is mounted at ```/mnt/usb/USB-A1```. Before you run the commands you should first check that is the case for your device. Insert your USB flash drive into your device and confirm the path with the following commands.

```bash
ls -las /mnt/usb/
```

The output should be something akin to the following. If not then you'll need to adjust the _script_ accordingly. Note the link (like a shortcut) ```USB-A1``` pointing to the actual mount point of the flash drive in ```/tmp```.

```bash
     0 drwxr-xr-x    2 root     root             0 Mar 26 17:26 .
     0 drwxr-xr-x    1 root     root             0 Nov  1 16:43 ..
     0 lrwxrwxrwx    1 root     root            20 Mar 26 17:26 USB-A1 -> /tmp/run/mountd/sda1
```

### Creating the backups
After inserting your USB flash drive into the device and checking its path as above you can run the following _script_ to backup your devices file system and compare the backups with the originals. While it's probably not neccessary to backup everything, it probably doesn't hurt either.

```bash
# Set USB path - change as appropriate if necessary 
USB="/mnt/usb/USB-A1"

# Display all the MTDs and save to a file on the flash for future reference
cat /proc/mtd | tee $USB/mtd.txt

# backup all the MTDs one at a time and check with file compare
for f in $(cat /proc/mtd | grep "mtd" | cut -d: -f1); \
do echo backing up $f; \
dd if=/dev/$f of=$USB/$f.dump; \
{ cmp /dev/$f $USB/$f.dump && echo ... $f done; } \
done

# create a tar.gz of /overlay/bank_2, which is useful for restoring on a file, by file basis
tar -cz -C /overlay -f $USB/bank_2.tar.gz bank_2
```

!!! hint ```mtd0``` (NAND.0) and ```mtd2``` (aka ```rootfs_data```  or ```userfs``` on older devices) will fail comparision, the latter probably because the bash history will change with each command run and this is stored here.

You should new have a backup of all the MTD filesystems on your device stored on the USB which you can now unmount.
