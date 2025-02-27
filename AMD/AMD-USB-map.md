With OpenCore I think it's about time we finally destroy some AMD myths, like how USB is just screwed on AMD and can't be mapped. Well that is false! And I will show you the path of enlightenment.

So why would I want to use this? Well couple reasons:
* Add missing USB ports that macOS didn't automatically add
* Remove unwanted devices like intel bluetooth conflicting with broadcoms
* Stability by removing USB port limit patches

# Backstory to USB on macOS

For the best explainer, please read Corp's [USB map guide](https://usb-map.gitbook.io/project/terms-of-endearment). For the lazy here's a super quick explainer:

* Apple intruduced a 15 port limit in OS X 10.11 El Capitan
* Each USB 3.0 port contains 2 personalities, 1 USB 2.0 personality and 3.0 personality
* So 8 USB 3.0 ports would be seen as 16 ports in macOS, thus going over the 15 port limit
* Most USB chipset controllers contain between 24 to 30 ports, most of which being internal
* XhciPortLimit patches this restriction out but is unstable and known for data corruption

# Getting started

So what you'll need to get started:
* Running macOS (this can be done in windows/linux though you'll need macOS to actually use the port map)
* [MaciASL](https://github.com/acidanthera/MaciASL/releases)
* [ProperTree](https://github.com/corpnewt/ProperTree) or some other plist editor
* [IORegistryExplorer](https://github.com/toleda/audio_ALCInjection/raw/master/IORegistryExplorer_v2.1.zip)
* [AMD-USB-Map.kext](https://github.com/khronokernel/Opencore-Vanilla-Desktop-Guide/tree/master/extra-files/AMD-USB-Map.kext.zip)
* `XhciPortLimit` enabled under `Kernel -> Quirks`
* Copy of your DSDT


# Gettting your DSDT

Couple different ways:
* [MaciASL](https://github.com/acidanthera/MaciASL/releases) -> Save as `System DSDT`, make sure the file format is ACPI Machine Language Binary
* F4 in Clover
   * DSDT can be found in `EFI/CLOVER/ACPI/origin`
* [SSDTTime](https://github.com/corpnewt/SSDTTime) for Linux
   * IOIIIO's fork of [SSDTTime](https://github.com/IOIIIO/SSDTTime) also supports windows DSDT dumping.
* [`acpidump.efi`](https://github.com/khronokernel/Opencore-Vanilla-Desktop-Guide/tree/master/extra-files/acpidump.efi.zip)
   * Add this to `EFI/OC/Tools` and in your config under `Misc -> Tools` then select this option in Opencore's picker

![](https://i.imgur.com/vHAomNm.png)

For those having issues running acpidump from the OpenCore picker can call it from the shell:

```
shell> fs0: //replace with proper drive

fs0:\> dir //to verify this is the right directory

   Directory of fs0:\
     01/01/01 3:30p   EFI
     
fs0:\> cd EFI\OC\Tools //note that its with forward slashes

fs0:\EFI\OC\Tools> acpidump.efi -b -n DSDT -z
```
You'll find that `DSDT.bat` is on the root of your EFI, reboot and rename the file to `DSDT.aml` and lets get cooking.

# Creating the map

So to start off, open IORegistryExplorer and find the USB controller you'd wish to map. For controllers, they come in some variations:

* XHC
* XHC0
* XHC1
* XHC2
* XHC3
* XHCI
* XHCX
* AS43
* PTXH (Commonly associated with AMD Chipset controllers)


The best way to find controllers is by searching for `XHC` and then looking at the results that come up, the parent of all the ports is the USB controller. Do note that many boards have multiple controllers but the port limit is per controller.

For today's example we'll be adding missing ports for the X399 chipset which has the identifier `PTXH`

![PTXH IOReg](https://i.imgur.com/wh7mMa4.png)

As you can see from the photo above, we're missing a shit ton of ports! Specifically ports POT3, POT4, POT7, POT8, PO12, PO13, PO15, PO16, PO17, PO18, PO19, PO20, PO21, PO22!

So how do we fix this? Well if you look in the corner you'll see the `port` value. This is going to be important to us when mapping

Next lets take a peak at our DSDT and check for our `PTXH` device:

![](https://i.imgur.com/ofYGYBS.png)
![](https://i.imgur.com/BZtkLl7.png)
All of our ports are here! So why in the world is macOS hiding them? Well there's a couple reasons but this being the main: Conflicting SMBIOS USB map

Inside the `AppleUSBHostPlatformProperties.kext` you'll find the USB map for most SMBIOS, this mean that that machine's USB map is forced onto your system. 

Well to kick out these bad maps, we gotta make a plugin kext. For us that's the [AMD-USB-Map.kext](https://github.com/khronokernel/Opencore-Vanilla-Desktop-Guide/tree/master/extra-files/AMD-USB-Map.kext.zip)

Now right click and press `Show Package Contents`, then navigate to `Contents/Info.plist`

![](https://i.imgur.com/Vfou3S1.png)
If the port values don't show in Xcode, right click and select `Show Raw Keys/Values`
![](https://i.imgur.com/ggsZw35.png)


So what kind of data do we shove into this plist? Well there's a couple sections to note:

* **Model**: SMBIOS the kext will match against
* **IONameMatch**: The name of the controlller it'll match against
* **port-count**: The last/largest port value that you want injected
* **port**: The address of the USB controller
* **UsbConnector**: The type of USB connector, which can be found on the [ACPI 6.3 spec, section 9.14](https://uefi.org/sites/default/files/resources/ACPI_6_3_final_Jan30.pdf)

UsbConnector types that we care about:
```
0: USB 2.0 Type A connector
3: USB 3.0 Type A connector
8: Type C connector - USB 2.0-only
9: Type C connector - USB 2.0 and USB 3.0 with Switch, flipping the device doesn't change ACPI port
10: Type C connector - USB 2.0 and USB 3.0 without Switch, flipping the device doesn't change ACPI port
255: Proprietary connector - For Internal USB ports like bluetooth
```
> How do I know which ports are 2.0 and which are 3.0?

Well the easiest way is grabbing a USB 2.0  and USB 3.0 device, then write down which ports are are what type from obsering IOReg. 

Now lets take this section:

```
Device (PO18)
{
   Name (_ADR, 0x12)  // _ADR: Address
   Name (_UPC, Package (0x04)  // _UPC: USB Port Capabilities
   {
        Zero, 
        0xFF, 
        Zero, 
        Zero
   })
}
```
For us, what matters is the `Name (_ADR, 0x12)  // _ADR: Address` as this tells us the location of the USB port. This value will be turned into our `port` value on the plist. Some DSDTs don't declare their USB address, for these situations we can see their IOReg properties.

![](https://i.imgur.com/9R6cab8.png)

**Reminder**: Don't drag and drop the kext, read the guide carefully. Rename `IONameMatch` value to the correct controller you're wanting to map and verify that the ports are named correctly to **your DSDT**. If you could drag and drop it and have it work for everyone there wouldn't be a guide ;p

Now save and add this to both your keytext (kext) folder and config.plist then reboot!

Now we can finally start to slowly remove unwanted ports and remove the XhciPortLimit quirk once you have 15 ports total or less.
# Port mapping on screwed up DSDTs

Something you may have noticed is that your DSDT is even missing some ports, like for example:

![AsRock B450 missing ports](https://i.imgur.com/xz3p0H4.png)

In this IOReg, we're missing HS02, HS03, HS04, HS05, etc. When this happens, we actually need to outright remove all our ports from that controller in our DSDT. What this will let us do is allow macOS to build the ports itself instead of basing it off of the ACPI. Save this modified DSDT.aml and place it in your EFI/OC/ACPI and specify it in your config.plist -> ACPI -> Add(note that DSDT.aml must be forst to work correctly)

# Fixing USB power on AMD

Something that users have noticed is that certain devices tend to break on certain AMD ports due to not suppling enough or correct power. Some of these devices include:

* Mics
* DACs
* Webcams
* Bluetooth Dongles

To fix this, we'll want to do is force USB power properties onto each USB controller by using [SSDT-USBX-AMD](https://github.com/khronokernel/Opencore-Vanilla-Desktop-Guide/blob/master/extra-files/SSDT-USBX-AMD.dsl). The 3 types of controllers most commonly found in AMD DSDTs are PTXH, XHC0 and AS43, **verify the ACPI path and that they exist in your DSDT before compiling this SSDT**. 

Credit to AlGrey and ydeng for their original work and findings here: [NATIVE RYZEN USB SUPPORT](https://amd-osx.com/forum/viewtopic.php?t=4986)
