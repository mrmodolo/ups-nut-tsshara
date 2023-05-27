# Nobreak Ts Shara on Ubuntu 20.04 with NUT

[NOBREAK UPS SENOIDAL UNIVERSAL 2200 #4222](https://tsshara.com.br/produto/nobreak-ups-senoidal-universal-2200-2/)

<img alt="Front panel" src="images/tsshara_front_panel.png" width="200" height="200"> <img alt="Rear panel" src="images/tsshara_rear_panel.png" width="200" height="200"> <img alt="Packaging" src="images/tsshara_packaging.png" width="200" height="200">

## Installation

```bash
sudo apt install nut
```

## Configuration

### UDEV

The manufacturer indicates in the specifications that the UPS supports "Intelligent Communication: with USB interface".

After unpacking and turning on the UPS, you can see a new USB device `STMicroelectronics Virtual COM Port`.

In my case with vendor id **0483** and product id **5740**.

```bash
Bus 003 Device 018: ID 0483:5740 STMicroelectronics Virtual COM Port
```

So for the NUT service to be able to use the new serial interface, it is necessary 
to create a rule for the UDEV to change permissions. 
It is also interesting to create a symbolic link so that the interface remains constant.

For this you need to create the file `/etc/udev/rules.d/99-ups-tsshara.rules`

```bash
SUBSYSTEM=="tty",ATTRS{idVendor}=="0483",ATTRS{idProduct}=="5740",GROUP="nut",OWNER="nut",MODE="0660",SYMLINK+="ttyTSSHARA"
```

```bash
sudo udevadm control --reload
sudo udevadm trigger
```

After that, it is necessary to verify that the settings worked and that 
the permissions were applied, in addition, the symbolic link must have been created.

```bash
ls -l /dev/ttyACM0
crw-rw---- 1 nut nut 166, 0 mai 27 16:29 /dev/ttyACM0

ls -l /dev/ttyTSSHARA
lrwxrwxrwx 1 root root 7 mai 27 14:49 /dev/ttyTSSHARA -> ttyACM0
```

### NUT

After installation it is necessary to change some files so that the NUT recognizes and can communicate with Ts Shara.
Below all the files that have been changed to allow the operation.

- [nut.conf](etc/nut/nut.conf)
- [ups.conf](etc/nut/ups.conf)
- [upsd.users](etc/nut/upsd.users)
- [upsmon.conf](etc/nut/upsmon.conf)
- [upssched.conf](etc/nut/upssched.conf)

Choosing the driver and its configuration is one of the most important steps.

In the etc/nut/ups.conf file we configure the driver and the port created by UDEV.

```toml
maxretry = 3

[tsshara]
  driver = "blazer_ser"
  port = "/dev/ttyTSSHARA"
  desc = "TS Shara"
``` 

## Verify Driver configuration

Start the driver.

```bash
sudo upsdrvctl start
```

## upsd configuration

Configure the file [upsd.users](etc/nut/upsd.users), I didn't have to change anything in the file
`/etc/nut/upsd.conf`.

```toml
[upsmon]
  password = "<PASSWORD>"
  upsmon master
  actions = SET
  instcmds = ALL
``` 

```bash
sudo systemctl start nut-server.service

sudo systemctl status nut-server.service

sudo systemctl enable nut-server.service

```

If the service goes up correctly, it is already possible to consult the status of the UPS.

```bash
upsc tsshara

Init SSL without certificate database
battery.charge: 100
battery.voltage: 27.00
battery.voltage.high: 26.00
battery.voltage.low: 20.80
battery.voltage.nominal: 24.0
device.mfr: TS SHARA 221011
device.model: Senoid  22
device.type: ups
driver.name: blazer_ser
driver.parameter.pollinterval: 2
driver.parameter.port: /dev/ttyTSSHARA
driver.parameter.synchronous: no
driver.version: 2.7.4
driver.version.internal: 1.57
input.current.nominal: 100.0
input.frequency: 60.0
input.frequency.nominal: 60
input.voltage: 208.0
input.voltage.fault: 208.0
output.voltage: 121.0
ups.beeper.status: enabled
ups.delay.shutdown: 30
ups.delay.start: 180
ups.firmware: V010201010
ups.load: 8
ups.mfr: TS SHARA 221011
ups.model: Senoid  22
ups.status: OL
ups.temperature: 24.0
ups.type: offline / line interactive
```

## Links and References

[Arch APC UPS](https://wiki.archlinux.org/title/APC_UPS)

[Arch Network UPS Tools](https://wiki.archlinux.org/title/Network_UPS_Tools)

[BLAZER_SER(8)](https://networkupstools.org/docs/man/blazer_ser.html)

[Notes on securing NUT](https://networkupstools.org/historic/v2.7.4/docs/user-manual.chunked/ar01s09.html)

[Network UPS Tools Overview](https://networkupstools.org/docs/user-manual.chunked/ar01s02.html#_configuring_and_using)

[Network UPS Tools (NUT)](https://networkupstools.org/)

[Utilizando NUT para controle de Nobreak TS SHARA com Raspberry](http://mylowtechstuff.blogspot.com/2018/02/utilizando-nut-para-controle-de-nobreak.html)

[Ubuntu 18.04 + Nobreak Ts shara USB](https://github.com/rbernardes/tsshara-ubuntu-server)

[How do I allow a non-default user to use serial device ttyUSB0?](https://askubuntu.com/questions/112568/how-do-i-allow-a-non-default-user-to-use-serial-device-ttyusb0)

[udev re-numbering when creating symlinks](https://unix.stackexchange.com/questions/79087/udev-re-numbering-when-creating-symlinks)

[UDEV rules, "NAME" variable not working](https://askubuntu.com/questions/920098/udev-rules-name-variable-not-working)

[SUSE Dynamic Kernel Device Management with udev](https://documentation.suse.com/sles/12-SP4/html/SLES-all/cha-udev.html)


