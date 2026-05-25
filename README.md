# Virtualhere

Virtualhere is a good bit of software but we all have the same problem with it. The the pricing of a non transferable license.

This repo is not meant to discuss that but instead to provide an alternative for most normal user situations as a result of it.

My setup is an Openwrt router configured as a dumb access point, acting as host with a usb hub connected to the single usb port the router has.

The client is a Windows 11 pc where I needed to use a Logitech unified dongle as well as a Gamesir controller + dongle.

## The host setup

This was done on a RT3200 running OpenWRT v25.12.4. The packet manager here is apk.

```sh
 OpenWrt recently switched to the "apk" package manager!

 OPKG Command           APK Equivalent      Description
 ------------------------------------------------------------------
 opkg install <pkg>     apk add <pkg>       Install a package
 opkg remove <pkg>      apk del <pkg>       Remove a package
 opkg upgrade           apk upgrade         Upgrade all packages
 opkg files <pkg>       apk info -L <pkg>   List package contents
 opkg list-installed    apk info            List installed packages
 opkg update            apk update          Update package lists
 opkg search <pkg>      apk search <pkg>    Search for packages
```

ssh to your device and run these commands. (depending on if you have opkg or apk)

opkg

```sh
opkg update
opkg install usbip usbip-server usbip-client usbutils
```

apk

```sh
apk update
apk add usbip usbip-client usbip-server usbutils
```

Do this to see the server is running

```sh
ps | grep '[u]sbipd'
```

Showing a result like this:

```sh
2231 root      1340 S    /usr/sbin/usbipd --ipv4 --ipv6
```

If that is fine, at this stage you want the hub plugged in with something you need to use plugged in, then do:

```sh
usbip list -l
```

You should see something like this:

```sh
root@rt3200:~# usbip list -l
 - busid 1-1.2 (xxxx:xxxx)
   Logitech, Inc. : Unifying Receiver (xxxx:xxxx)

 - busid 1-1.1 (xxxx:xxxx)
   ASIX Electronics Corp. : AX88179 Gigabit Ethernet (xxxx:xxxx)

 - busid 1-1.3 (xxxx:xxxx)
   unknown vendor : unknown product (xxxx:xxxx)
```

> [!TIP]
> I used one of these
> https://www.amazon.co.uk/dp/B09NNHF6MK with a usb-c to usb adapter because the router does not have a usb-c connector.
>
> If you do this they seem to only work when one way. The usb-c is not reversible with an adapter. If `usbip list -l` returns an empty result try reversing the usb-c connector.

Once you see the devices listed like above, you will see that each port (including things like ethernet) will have a fixed address like `1-1.1`

In my example is it `busid 1-1.2` for my logitech dongle and `busid 1-1.3` for my gamesir dongle.

Once this part is ok, make them hot-pluggable.

This command will make so that when devices are plugged in or unplugged from the hub they will be bound or unbound automatically.

```sh
cat > /etc/hotplug.d/usb/99-usbip <<'EOF'
#!/bin/sh
case "$ACTION" in
    add|bind) usbip bind -b "$DEVICENAME" ;;
    remove) usbip unbind -b "$DEVICENAME" ;;
esac
EOF
```

Make it executable

```sh
chmod +x /etc/hotplug.d/usb/99-usbip
```

To make them bind at startup you need to do this:

This will add the command before the final `exit 0` of the `/etc/rc.local` file.

> [!TIP]
> All you need to do is make sure the `1-1.1 1-1.2 1-1.3 1-1.4` par matches your device id numbers. For each id listed, it will bind it at startup. You cna just use the ID's you need.

```sh
sed -i '/^exit 0$/i\
sleep 10\
for busid in 1-1.1 1-1.2 1-1.3 1-1.4; do\
    usbip list -l | grep -q "busid $busid" && usbip bind -b "$busid"\
done\
' /etc/rc.local
```

This configuration will mean your usb hub devices are automatically bound at startup/reboot and automatically bound/unbound on adding or removing devices to and from the hub.

Nothing else to do here. Client side time.

## The client setup

I found this to be the best option for windows.

https://github.com/vadimgrn/usbip-win2 - https://github.com/vadimgrn/usbip-win2/releases/latest

At the time of this guide I used this version

https://github.com/vadimgrn/usbip-win2/releases/download/v.0.9.7.7/USBip-0.9.7.7-x64.exe

Download and install it.

When it is open:

Top right, set your server host IP. the port 3240 is the default.

Click add devices. Your bound items should be listed.

Right click the device you want to attach to and attach it.

It will now have a port shown for the device.

That is it, it should just work now.

I had no issues so far.
