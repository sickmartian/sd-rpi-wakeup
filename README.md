# Wake up your Steam Deck remotely

## What and why?

This guide helps you setup a 1st gen Steam Deck (SD), placed in a Steam Docking Station (SDS), to wake up remotely using a Raspberry Pi (RPi) with OTG capabilities by simulating a key press. This repository consists of an overview of the process and [an RPi OS image](https://github.com/sickmartian/sd-rpi-wakeup/releases) to use on the RPi, as well as links to other guides and sources used to put this together.

This repository is open to contributions for alternative steps, troubleshooting information, and information on differences that might apply to achieve this with other devices, like a RPi 4.

The motivation is a SD that sits far away, inside of a cabinet, and is used as a media center, so turning it on an off happens all the time. 2nd+ gen Steam Decks support waking the device remotely, but the 1st gen never got this ability unless you happen to have a special dock or a Steam Controller. Even so, the Steam Controller solution is not perfect as it requires batteries which you have to buy or recharge, which is very annoying.

## Demo



https://github.com/sickmartian/sd-rpi-wakeup/assets/492246/e8fb095b-c3b9-4aff-95a4-d5f20aa21538



## Hardware needed

- Steam Deck
- Steam Docking Station (other docks may work too)
- Raspberry Pi Zero 2 W (or Raspberry Pi 4, or any RPi that supports OTG should work but I this is only tried on RPi Zero 2 W)
  - If you don't have one, you can find the cheapest RPi closest to you with [rpilocator](https://rpilocator.com/). For reference, I got an [RPi Zero 2 W](https://rpilocator.com/?cat=PIZERO2) with SKU `SC0510` (no headers)
- MicroSD Card, to install the OS on the RPi
- [USB A to micro USB male to male cable](./micro-usb-to-usb-a.jpg), or a combination of [USB-A to micro USB female to male OTG cable and a USB-A to USB-A male to male cable](./otg-and-usba-usba-combo.jpg)
- Wi-Fi network the RPi supports (RPi Zero 2 W only sees my 2.4 GHz network, but my RPi 3 B+ can see the 5 GHz one, so I assume the RPi 4 can too)
- Some way to read the SD card
  - e.g. the PC has a slot for SD and the MicroSD card comes with an adapter, or a USB to MicroSD converter or similar
- PC - ideally with MacOS or Linux
  - In case you don't want to use the pre-packaged OS image it should be powerful enough to run Docker and compile the Linux kernel. I'm using a MacBook Pro with Apple Silicon and will be linking to some instructions I found useful [below](#optional---manual-setup).
- (Optional) Power source for the RPi
  - I was able to use the SDS to power the RPi Zero 2 W, but that might not work with all RPis or all docks
  - It may also be useful to repurpose the RPi in the future
- (Optional) Case for the RPi
  - Depends on the RPi model. Mine came with the RPi
- (Optional) HDMI mini to HDMI cable or adapter
  - Can help with troubleshooting, or if you want to repurpose the RPi later on
- (Optional) Device to run bash scripts that wake or make the SD sleep
  - Can be the PC, but I use an Android phone with Termux for convenience

## Software needed

- Terminal - of which there is usually one preinstalled in your PC already
  - Since I'm on Mac, I use [iTerm2](https://iterm2.com/)
- [OpenSSH](https://www.openssh.com/) - usually preinstalled
- [git](https://git-scm.com/) - usually preinstalled
- [rpi-imager](https://www.raspberrypi.com/software/)
- [pre-packaged OS image](https://github.com/sickmartian/sd-rpi-wakeup/releases) - get the latest from the releases section
- (Optional) [termux](https://github.com/termux/termux-app#installation)
- (Optional) [Docker Desktop](https://www.docker.com/products/docker-desktop/)

## General idea

Setting things up is a multi-step process. It all gets a bit more complex since, at the time of writing, the official RPi Linux kernel [has a bug](https://github.com/raspberrypi/linux/issues/3977), which makes it so that even if the RPi is recognized as a keyboard, it doesn't think it can wake the device it's connected to.

Luckily, [mdavaev](https://github.com/mdevaev) from the PiKVM project commented on the bug thread with a patch they created, and [jlian](https://github.com/jlian) provides step by step instructions on how to compile the kernel while applying the patch, and how to copy the new kernel into the RPi system.

For your convenience, you can download the [pre-packaging an OS image](https://github.com/sickmartian/sd-rpi-wakeup/releases) with the patch applied, so all you really would need to do is download a big file and install it on your RPi as an OS.

If you wish, the [(Optional) Manual Setup](#optional---manual-setup) section points to the instructions from jilian, so you can build the OS image yourself. This is helpful if you already have an RPi with data you don't want to lose, if you don't want the same OS I'm using, don't trust a random internet person, etc.

With the patched OS out of the way, first thing we are going to do is gather the SSH identities from the computers we want to connect to the RPi (the PC to prepare everything + whatever you want to use to wake the SD). Then, we are going to prepare the MicroSD card for the RPi in such a way we can connect with those devices, connect everything together, find and connect to the RPi from your device of choice, and do a test.

## SSH Keys gathering

Open up the terminal and let's check if you have an .ssh identity created

```bash
ls ~/.ssh/*.pub
```

If there are any files, like `/Users/<your_user>/.ssh/id_rsa.pub` or `/Users/<your_user>/.ssh/id_ed25519.pub` then you are all set.

**IF AND ONLY IF** you do not have `.pub` files, you must create a new one. To do so, run `ssh-keygen` and follow the wizard.

```bash
# only run me if you do not have keys already!
$ ssh-keygen
```

You can just keep pressing Enter for most of it.

Note that the passphrase _must_ be left blank, if you want to achieve a 'single command' solution, otherwise you will be required to enter the passphrase to turn the SD on, which will make the Termux configuration shown on the demo no work. It's safe to leave this blank if your PC isn't shared with anyone, and your drives are encrypted.

The `ssh-keygen` output should (roughly) look like this:

```text
➜  ~ ssh-keygen
Generating public/private ed25519 key pair.
Enter file in which to save the key (/Users/<your_user>/.ssh/id_ed25519): /Users/<your_user>/.ssh/id_ed25519
Enter passphrase (empty for no passphrase):
Enter same passphrase again:
Your identification has been saved in /Users/<your_user>/.ssh/id_ed25519
Your public key has been saved in /Users/<your_user>/.ssh/id_ed25519.pub
The key fingerprint is:
SHA256:DNd0HTC8sDw6CslmFO+S2+tV0RyylSj1Kws7cf+XFxI <your_user>@<your_pc>.local
The key's randomart image is:
+--[ED25519 256]--+
|         .+o*+.. |
|    .   .ooOoo.  |
|     o. .o+o+.   |
|    . .+  +..E   |
|   o +  S.+.. .  |
|    O . o* + . . |
|   o = .+.. . . o|
|    . o. .   . .o|
|     .o.      ...|
+----[SHA256]-----+
```

Once done, save the public key generated on some temporal file. You can get it by doing:

```bash
cat ~/.ssh/id_ed25519.pub
```

Or if you had an `id_rsa.pub` file:

```bash
cat ~/.ssh/id_rsa.pub
```

If you want another device to be able to turn on the SD, you can perform the same steps on them, and save the public key for those too.

## Using the image

In your PC, insert the MicroSD card, open up `rpi-imager`:

![RPi Imager - Start](./1-rpi-imager.png)

1. Click on **CHOOSE DEVICE**, and select the device you actually have.
2. Click on **CHOOSE OS**, and chose the `rpi-sd-wake-no-config.img` file from this repository's [latest release](https://github.com/sickmartian/sd-rpi-wakeup/releases), unless you want to go the Manual setup route, on which case you get to choose an OS yourself.
3. Click on **CHOOSE STORAGE**, and select the MicroSD card

![RPi Imager - Image set](./2-rpi-imager.png)

Now click on **NEXT**, and a popup should show up:

![RPi Imager - Popup 1](./3-rpi-imager.png)

Click on **EDIT SETTINGS**, and in the "GENERAL" tab fill up:

- **Hostname for the RPi (`rpi_hostname`)**: make sure it's unique in your network and you write it down, as we are going to use it later
- **Username (`rpi_username`) and Password**: while the password won't normally be used, write them down as well. As always, make the password very complex and unique
- **SSID** and **Password** of your Wi-Fi network: note that some RPi models like the RPi Zero 2 W only support 2.4GHz networks

![RPi Imager - General Tab](./4-rpi-imager.png)

Then go to the "SERVICES" tab, and make sure `Enable SSH` is checked and `Allow public-key authentication only` is selected. Below under "Set authorized_keys for '...'", paste the ssh public keys we gathered from the `.pub` files on the previous step.

![RPi Imager - Services Tab](./5-rpi-imager.png)

Once done, click on **SAVE**, and another popup should show up:

![RPi Imager - Popup 2](./6-rpi-imager.png)

Confirm you are not erasing something important from the MicroSD card, and click on **YES**.

The program should now start copying the image to the MicroSD card. This could take a while.

After writing and verification are complete, the MicroSD card should be ejected automatically. To make sure of that, open up Finder and check that there are no extra items like `bootfs` under "Locations" on the the left sidebar. If there are, just click on the icon (⏏️) to eject them manually, and remove the MicroSD card from your PC.

![Finder left panel with bootfs](./finder-eject.png)

### Connect the RPi

Insert the MicroSD card into the RPi. If you have a case for the RPi, you can put the RPi into the case, as we won't be removing the MicroSD card anymore.

Insert the Micro USB end of the male to male cable into the port marked as **USB** of the RPi, and the USB A end into one of the slots of the SDS.

The RPi LED should light up:

![RPi connected and powered](./rpi-on.png)
> The LED on my RPi is that green light on the top left, indicating the RPi is receiving power.

After waiting some minutes for the RPi to boot up with our image, we should be able to find it from our PC.

In MacOS and Linux, finding the RPi is pretty straightforward as `<our_selected_hostname>.local` [should normally point to it](https://avahi.org/) if the Wi-Fi setup was done correctly, and both your RPi and your PC are on the same network.
For other OSs, you will have to check with the router for the IP of the RPi, usually in a section about DHCP leases.

We can try login in from our PC's terminal by using the command below. Replace `<rpi_username>` and `<rpi_domain_or_ip>` with your own values (e.g. in my case this would be `sickmartian` and `rpiz.local` respectively):

```bash
ssh <rpi_username>@<rpi_domain_or_ip>
```

![How a logged ssh session looks like](./ssh-first-time.png)

This should drop you into a shell session inside the RPi.

The pre-packaged OS image comes preconfigured with the scripts required to wake up the SD. If you are **not** using the pre-packaged OS image, refer to the [(Optional) Manual Setup](#optional---manual-setup) section to set that up.

Once you already have the scripts, we can do a test:

1. First, make sure to turn on the SD _with the RPi connected_
2. Then, use a controller to put it to sleep
3. Now we can try to wake it up from this console:

    ```bash
    sudo ./wake-up-sd.sh
    ```

After a second or so, the script should send a key to the SD, waking it up!

You can now exit the ssh session by typing `exit` and pressing Enter.

From now on, from your PC you can run a single line to wake up the SD. Once again, replace `<rpi_username>` and `<rpi_domain_or_ip>` like before:

```bash
ssh <rpi_user>@<rpi_domain_or_ip> -x "sudo ./wake-up-sd.sh"
```

## Optional - Manual setup

If you don't want to use the [pre-packaged OS image](https://github.com/sickmartian/sd-rpi-wakeup/releases), you can get there yourself with the steps below.

> **NOTE:** there is no handholding here, no mention on how to edit files, go to root, etc, if you need this I assume you have some Linux experience already

Follow steps (1), (2) and (3) of [this guide](https://randomnerdtutorials.com/raspberry-pi-zero-usb-keyboard-hid/) to set the RPi as an USB Gadget. That done, we are 80% of the way there, and all that's left is the other 80%.

You could follow the next steps from the guide above to create a python script and send key presses that way, but then we hit [this issue](https://github.com/raspberrypi/linux/issues/3977) that doesn't let us wake the SD.

Following the guidance of that thread, we should change the content of `/usr/bin/isticktoit_usb` to include the `wakeup_on_write` and the `bmAttributes`, leaving the 'functions' section as follows:

```text
# Add functions here
mkdir -p functions/hid.usb0
echo 1 > functions/hid.usb0/protocol
echo 1 > functions/hid.usb0/subclass
echo 8 > functions/hid.usb0/report_length
echo -ne \\x05\\x01\\x09\\x06\\xa1\\x01\\x05\\x07\\x19\\xe0\\x29\\xe7\\x15\\x00\\x25\\x01\\x75\\x01\\x95\\x08\\x81\\x02\\x95\\x01\\x75\\x08\\x81\\x03\\x95\\x05\\x75\\x01\\x05\\x08\\x19\\x01\\x29\\x05\\x91\\x02\\x95\\x01\\x75\\x03\\x91\\x03\\x95\\x06\\x75\\x08\\x15\\x00\\x25\\x65\\x05\\x07\\x19\\x00\\x29\\x65\\x81\\x00\\xc0 > functions/hid.usb0/report_desc
echo 1 > functions/hid.usb0/wakeup_on_write
ln -s functions/hid.usb0 configs/c.1/
echo 0xa0 > configs/c.1/bmAttributes
# End functions
```

Now, for this to work, we also need to [apply the kernel patch from mdevaev](https://github.com/raspberrypi/linux/issues/3977#issuecomment-1200368214) which you can do following [jilian instructions](https://github.com/jlian/linux-kernel-cross-compile)

Once that's done, you can create the `wake-up-sd.sh` script with the following content:

```bash
#!/bin/bash
# Send keyboard power key
echo -ne "\0\0\x66\0\0\0\0\0" > /dev/hidg0
# Release
echo -ne "\0\0\0\0\0\0\0\0" > /dev/hidg0
```

Give it execute permissions, and run it to wake up the SD:

```bash
chmod +x wake-up-sd.sh
sudo ./wake-up-sd.sh
```

## Optional - Allow the Steam Deck to suspend w/o password

This is so we can get a single one-liner that doesn't require a password to suspend the SD. You can skip this is you are fine suspending it from a controller, this is useful for controlling the SD from an Android device via [KDE Connect](https://kdeconnect.kde.org/) for example.

First, if you didn't already, set up ssh on your SD. You can follow a guide [like this one](https://pimylifeup.com/steam-deck-ssh/).

Next steps assume you have the IP of the SD (`<sd_ip>`) as well as the password for the `deck` user, and that this user is able to run commands as root.

First, ssh from the PC (like we did for the RPi) with username and password, replacing `<sd_ip>` with the IP of the SD.

```bash
ssh deck@<sd_ip>
# you will be asked for a password and land in:
# (deck@steamdeck ~)$
```

Our first objective now is to be able to connect to the SD without a password, for this we want to authorize the same public keys we wanted to use in the RPi on the SD.
Let's start by creating the `ssh` config directory, if it doesn't exist already:

```bash
mkdir -p ~/.ssh
```

Then for each public key, you do the following (make sure to replace `<your_public_key>`):

```bash
echo '<your_public_key>' >> ~/.ssh/authorized_keys
```

Once that's done, let's try it out, getting out of ssh first...

```bash
exit
# should land you to your normal session
```

... and connecting back again (make sure to replace `<sd_ip>`):

```bash
ssh deck@<sd_ip>
```

If we did things properly, this shouldn't ask you for a password.

Now that we are back on the SD, our second objective is being able to suspend.
This is done via `systemctl suspend`, but if you try that, you will get a prompt requesting the password for your `deck` user:

![SD Requesting password to suspend](./systemctl-suspend-password.png)

To remove this, you need to add a rule to bypass this check for our `deck` user:

```bash
sudo su # should ask you for the password and land you into (B)(root@steamdeck deck)#
cd /etc/polkit-1/rules.d/
nano 85-bypass.rules
```

That last command should land you into an editor. There, paste the following:

```c++
polkit.addRule(function(action, subject) {
        if (action.id == "org.freedesktop.login1.suspend" &&
                subject.isInGroup("deck")) {
                return polkit.Result.YES;
        }
});
```

And press `Ctrl+X` to exit, `Y` to save, and `ENTER` to confirm the filename.

![Nano editor with rule pasted](./rule-to-bypass-password.png)

Now you do `exit` to leave the root shell and get back into `(deck@steamdeck ~)$`. Here, we get to try out the suspend command again `systemctl suspend`, which should suspend your SD without requesting any password!

Doing `exit` again takes you back to PC's shell. Here, we can try waking up the SD and making it sleep, make sure to replace `<rpi_username>`, `<rpi_hostname>` and `<sd_ip>`:

```bash
ssh <rpi_username>@<rpi_hostname>.local -x "sudo ./wake-up-sd.sh"
# wait some seconds for the RPi to wake up
ssh deck@<sd_ip> -x "systemctl suspend"
```

## Optional - Create a remote-control with Termux

To complete the setup as demonstrated, you will need [Termux](https://termux.dev/en/) and [Termux:Widget](https://f-droid.org/en/packages/com.termux.widget/) which should be installed via [F-Droid](https://f-droid.org/en/).

Once installed in your Android device, open it up and run the steps from [SSH Keys gathering](#ssh-keys-gathering) to generate and get your SSH public key. Send that public key to your PC in some way (e.g. emailing it yourself, using a messaging app, etc.).

Now, from the PC, log into the RPi and authorize it (make sure to replace `<rpi_username>`, `<rpi_hostname>`, and `<termux_public_key>`):

```bash
ssh <rpi_username>@<rpi_hostname>.local
echo '<termux_public_key>' >> ~/.ssh/authorized_keys
exit
```

And do the same for the SD (also replace `<sd_ip>`):

```bash
ssh deck@<sd_ip>
echo '<termux_public_key>' >> ~/.ssh/authorized_keys
exit
```

Now we can run the one-liners we used in our PC on Termux, and the SD should suspend and wake up.

To make this one-liners work from the Termux widget, we have to create them as scripts in `.shortcuts/tasks`. You should be able to copy & paste the following into Termux (make sure to replace `<rpi_username>`, `<rpi_hostname>` and `<sd_ip>`):

```bash
cd ~ # make sure we are in the proper dir
mkdir -p .shortcuts/tasks # creates the directory
touch .shortcuts/tasks/sd-sleep.sh # creates the file
chmod +x .shortcuts/tasks/sd-sleep.sh # makes it executable
echo '#!/bin/bash' > .shortcuts/tasks/sd-sleep.sh # it's a bash script
echo 'date -Iseconds >> ~/sd-sleep.log' >> .shortcuts/tasks/sd-sleep.sh # log time
echo 'ssh deck@<sd_ip> -x "systemctl suspend" >> ~/.sd-sleep.log 2>&1' >> .shortcuts/tasks/sd-sleep.sh # suspend and log output and errors
touch .shortcuts/tasks/sd-wake-up.sh # creates the file
chmod +x .shortcuts/tasks/sd-wake-up.sh # makes it executable
echo '#!/bin/bash' > .shortcuts/tasks/sd-wake-up.sh # it's a bash script
echo 'date -Iseconds >> ~/sd-wake-up.log' >> .shortcuts/tasks/sd-wake-up.sh # log time
echo 'ssh <rpi_username>@<rpi_hostname>.local -x "sudo ./wake-up-sd.sh" >> ~/.sd-wake-up.log 2>&1' >> .shortcuts/tasks/sd-wake-up.sh # wake and log output and errors
```

Then add the Termux widget on your home screen. After pressing the refresh (🔃) button, the `sd sleep.sh` and `sd wake up.sh` scripts should show up, and you should be able to press them to run them.

![Termux widget with scripts](./termux-widget.png)

## Other considerations

You might want to update the RPi packages via:

```bash
sudo apt update
sudo apt upgrade
```

If you are going to use a PC to control your SD and you are mostly working in desktop mode, I recommend to use [Barrier](https://github.com/debauchee/barrier) to share keyboard and mouse.
I had issues with discovery of the server from the SD, but once it starts properly, it's solid.

If you are going to use your Android phone to control your SD, I highly recommend [KDE Connect](https://kdeconnect.kde.org/).
For me it was already installed in the SD, requires very small to no configuration, and works great.

## Links and resources

- [rpilocator](https://rpilocator.com/)
- [The awesome PiKVM](https://pikvm.org/)
- [Waking up via RPi as keyboard issue](https://github.com/raspberrypi/linux/issues/3977)
- [jlian's guide for compiling the kernel with the patch](https://github.com/jlian/linux-kernel-cross-compile)
- [Original guide used by jlian for 64 bits](https://github.com/geerlingguy/raspberry-pi-pcie-devices/tree/master/extras/cross-compile)
- [Information about USB Gadget](https://docs.kernel.org/6.2/usb/gadget_hid.html)
- [More readable list of keys to send via the RPi](https://gist.github.com/MightyPork/6da26e382a7ad91b5496ee55fdc73db2)
- [Helper used to create the image](https://github.com/Drewsif/PiShrink)
- [Barrier](https://github.com/debauchee/barrier)
- [KDE Connect](https://kdeconnect.kde.org/)
