# Nobreak Ts Shara on Ubuntu 20.04 with NUT

[NOBREAK UPS SENOIDAL UNIVERSAL 2200 #4222](https://tsshara.com.br/produto/nobreak-ups-senoidal-universal-2200-2/)

<img alt="Front panel" src="images/tsshara_front_panel.png" width="200" height="200"> <img alt="Rear panel" src="images/tsshara_rear_panel.png" width="200" height="200"> <img alt="Packaging" src="images/tsshara_packaging.png" width="200" height="200">

## Installation

```bash
sudo apt install nut nut-client nut-server
```

## Configuration

### UDEV

The manufacturer indicates in the specifications that the UPS supports "Intelligent Communication: with USB interface".

After unpacking and turning on the UPS, you can see a new USB device `STMicroelectronics Virtual COM Port`.

In my case with vendor id **0483** and product id **5740**.

```bash
Bus 003 Device 018: ID 0483:5740 STMicroelectronics Virtual COM Port
```

```bash
udevadm info -a -n /dev/ttyACM0
```

```
...
    ATTRS{idProduct}=="5740"
    ATTRS{idVendor}=="0483"
    ATTRS{manufacturer}=="STMicroelectronics"
    ATTRS{product}=="STM32 Virtual ComPort"
    ATTRS{serial}=="00000000001A"
...

```

So for the NUT service to be able to use the new serial interface, it is necessary 
to create a rule for the UDEV to change permissions. 
It is also interesting to create a symbolic link so that the interface remains constant.

For this you need to create the file `/etc/udev/rules.d/99-ups-tsshara.rules`

It is important to remember that we can have other devices with the same virtual port controller, 
so it is important to choose a set of attributes that allows differentiating multiple UPSs even 
if they are from the same model and manufacturer.

So I chose three attributes that seem to differentiate each port connected to the computer.

- ATTRS{idProduct}=="5740"
- ATTRS{idVendor}=="0483"
- ATTRS{serial}=="00000000001A"

The symbolic link ensures that the port name is always the same, which will facilitate NUT configuration.

```bash
SUBSYSTEM=="tty",ATTRS{idVendor}=="0483",ATTRS{idProduct}=="5740",ATTRS{serial}=="00000000001A",GROUP="nut",OWNER="root",MODE="0664",SYMLINK+="ttyTSSHARA0"
```

```bash
sudo udevadm control --reload
sudo udevadm trigger
```

After that, it is necessary to verify that the settings worked and that 
the permissions were applied, in addition, the symbolic link must have been created.

```bash
ls -l /dev/ttyACM0
crw-rw---- 1 root nut 166, 0 mai 27 16:29 /dev/ttyACM0

ls -l /dev/ttyTSSHARA0
lrwxrwxrwx 1 root root 7 mai 27 14:49 /dev/ttyTSSHARA0 -> ttyACM0
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
  port = "/dev/ttyTSSHARA0"
  desc = "TS Shara"
  default.battery.voltage.high = "26.00"
  default.battery.voltage.low = "20.80"
  default.battery.voltage.nominal = "24.00"
  runtimecal = "3600,100,7200,50"  
``` 

## Verify Driver configuration

Start the driver.

```bash
sudo upsdrvctl start
```

## UPSD configuration

Configure the file [upsd.users](etc/nut/upsd.users), I didn't have to change anything in the file
`/etc/nut/upsd.conf`.

```
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

**$ upsc tsshara**

```bash

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
driver.parameter.port: /dev/ttyTSSHARA0
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
## UPSMON Configuration

Add the lines below to the file `/etc/nut/upsmon.conf`

```
MONITOR tsshara@localhost 1 upsmon "<PASSWORD>" master
NOTIFYFLAG ONBATT SYSLOG+WALL+EXEC
NOTIFYFLAG ONLINE SYSLOG+WALL+EXEC
```

```
MINSUPPLIES 1
SHUTDOWNCMD "/sbin/shutdown -h +0"
POLLFREQ 5
POLLFREQALERT 5
HOSTSYNC 15
DEADTIME 15
POWERDOWNFLAG /etc/killpower
RBWARNTIME 43200
NOCOMMWARNTIME 300
FINALDELAY 5
MONITOR tsshara@localhost 1 upsmon "<PASSWORD>" master
NOTIFYFLAG ONBATT SYSLOG+WALL+EXEC
NOTIFYFLAG ONLINE SYSLOG+WALL+EXEC
```

It is necessary to start and activate the service.

```bash
sudo systemctl start nut-monitor.service

sudo systemctl enable nut-monitor.service
```

## Enable NUT after reboot

```bash

sudo systemctl nut-client.service   enabled
# sudo systemctl nut-driver.service   can not be enabled
sudo systemctl nut-monitor.service  enabled
sudo systemctl nut-server.service   enabled
sudo systemctl nut-logger.service   enabled

```

## "Oh My ZSH!" and Powerlevel10k

I use ["Oh My ZSH!"](https://ohmyz.sh/) in my day to day with the [Powerlevel10k](https://github.com/romkatv/powerlevel10k) 
theme and I decided to try to create a custom prompt with some UPS information.

<img alt="prompt_my_ups()" src="images/prompt_my_ups.png" width="272" height="38">

Below is the function I added to the `~/.p10k.zsh` file and the configuration to add the new prompt.

```zsh
  typeset -g POWERLEVEL9K_RIGHT_PROMPT_ELEMENTS=(
    # =========================[ Line #1 ]=========================
    ...
    my_ups                  # show UPS input voltage and battery charge if no power
  )

  typeset -g POWERLEVEL9K_MY_UPS_BACKGROUND=237
  # Aneel Resolution No. 505/2001 – National Electric Energy Agency
  # establishes limits of 201 to 231 V (for phase-to-neutral voltage) for the
  # electricity supply by concessionaires in Brazil,
  # considering nominal value of 220/380V, three-phase.
  # Maximum: 231V
  # Average: 216V
  # Minimum: 201V  
  function prompt_my_ups() {
    integer input_voltage="$(upsc tsshara input.voltage 2>/dev/null)"
    # No power
    if (( input_voltage <= 1 )); then
      integer battery_charge="$(upsc tsshara battery.charge 2>/dev/null)"
      p10k segment -f red -i '🔌' -t "${battery_charge}🔋"
      return 0
    fi
    if (( input_voltage < 201 || input_voltage > 231 )); then
      p10k segment -f red -i '🔌' -t "${input_voltage}⚡"
    elif (( input_voltage < 209 || input_voltage > 220 )); then
      p10k segment -f yellow -i '🔌' -t "${input_voltage}⚡"
    else
      p10k segment -f green -i '🔌' -t "${input_voltage}⚡"
    fi
  }
```
## Other Models

[Nobreak 1.5Kva Ts Shara 4438 Ups 8 Tomadas E.Biv S.Chaveado](https://tsshara.com.br/produto/nobreak-ups-senoidal-universal-1500va-novo-design/?gclid=Cj0KCQjw98ujBhCgARIsAD7QeAg8e9vtDRaCtPpG8RoNwE9VNdTUkajAgu34aJDS-Z20pqZvDqlBCZgaAtd5EALw_wcB)

I had the opportunity to also install and test the Ts Shara 4438 model on 
Ubuntu 22.04 and it worked perfectly using the same BLAZER_SER driver.

I started noticing some messages like 'stale' when trying to get the status of the UPS, 
after some research I found the link [NUT & CyberPower UPS](https://nmaggioni.xyz/2017/03/14/NUT-CyberPower-UPS/) 
and decided to test these settings on the equipment. 
Let's see if the result will be successful...

Update Jun 2023: I still have stale errors so I'm going to test the driver [nutdrv_qx](https://networkupstools.org/docs/man/nutdrv_qx.html) 
to see if the problem goes away

**/etc/nut/ups.conf**

```toml
[tsshara]
  driver = "nutdrv_qx"
  port = "/dev/ttyTSSHARA0"
  default.battery.voltage.high = "26.00"
  default.battery.voltage.low = "20.80"
  default.battery.voltage.nominal = "24.00"
  runtimecal = "1800,100,3600,50"
  desc = "TS Shara"
  protocol = "megatec"
  pollinterval = 15
```  

```
/etc/nut/ups.conf
[tsshara]
  ...
  pollinterval = 15

/etc/nut/upsmon.conf
  ...
  DEADTIME 25

/etc/nut/upsd.conf  
  ...
  MAXAGE 25
```

Still having stale errors in the driver. I found one more article to 
test [Restart nut-driver when data stale, usb device keeps changing](https://unix.stackexchange.com/questions/333351/restart-nut-driver-when-data-stale-usb-device-keeps-changing)

```
ACTION=="add", \                                                                                       
SUBSYSTEM=="usb", \                                                                                    
ATTR{idVendor}=="0665", ATTR{idProduct}=="5161", \                                                     
SYMLINK+="powerwalkerups", \                                                                           
MODE="0660", GROUP="nut", \                                                                            
RUN+="/bin/systemctl restart nut-driver"
```

[13. What’s this about data stale?](https://networkupstools.org/docs/FAQ.html#_what_8217_s_this_about_emphasis_data_stale_emphasis)

It means your UPS driver hasn’t updated things in a little while. upsd refuses to serve up data that isn’t fresh, 
so you get the errors about staleness.

If this happens to you, make sure your driver is still running. Also look at the syslog. 
Sometimes the driver loses the connection to the UPS, and that will also make the data go stale.

This might also happen on certain virtualization platforms. If you cannot reproduce the problem on a physical machine, 
please report the bug to the virtualization software vendor.

If this happens a lot, you might consider cranking up DEADTIME in the upsmon.conf to suppress some of the warnings 
for shorter intervals. Use caution when adjusting this number, since it directly affects how long you run on 
battery without knowing what’s going on with the UPS.

Note: some drivers occasionally need more time to update than the default value of MAXAGE (in upsd.conf) allows. 
As a result, they are temporarily marked stale even though everything is fine. This can happen with MGE Ellipse 
equipment — see the mge-shut or usbhid-ups man pages. 
In such cases, you can raise the value of MAXAGE to avoid these warnings; try a value like 25 or 30.

## UPSLOG

Settings for storing log status and rotation

|   Flag  |                                                                                                                                     Description                                                                                                                                    |
|:-------:|:----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------:|
| OL      | Online. Power is provided by the wall outlet (line).                                                                                                                                                                                                                               |
| OB      | On Battery. Power is provided by the UPS battery and inverter.                                                                                                                                                                                                                     |
| LB      | Low Battery. The estimated charge capacity of the battery, and thus the estimated runtime, are below a threshold. This can occur when on-line or on-battery. When also on-battery, a LB condition generally causes NUT to initiate shutdown.                                       |
| HB      | High Battery. The battery voltage is above a threshold. This is a relatively unusual problem. Typically it would indicate a bad battery or a malfunctioning UPS.                                                                                                                   |
| RB      | Replace Battery. The UPS has decided the battery is no-good. This may be based on time in service, number of charge cycles, the results of a self-test, or anything else.                                                                                                          |
| CHRG    | Battery Charging. The battery is not fully charged, so the UPS is charging it. My UPS never indicates this to NUT.                                                                                                                                                                 |
| DISCHRG | Battery Discharging. Self-explanatory, but I don't know the difference between this and "OB". My UPS never indicates this to NUT.                                                                                                                                                  |
| BYPASS  | Bypass Active. The power protection (inverter, regulators, etc.) in the UPS has been bypassed, and the load is electrically connected directly to the wall outlet. Typically occurs if the UPS detects an internal fault. Most small UPSes, mine included, lack bypass capability. |
| ALARM   | The UPS is indicating a trouble condition, not further specified. This could be low or bad battery, high temperature, an internal fault, UPS on fire, etc.                                                                                                                         |
| TEST    | Self Test. Most UPSes can perform a brief self-test, periodically and/or on command. This indicates such a test is in progress.                                                                                                                                                    |
| CAL     | Runtime Calibration. Some UPSes can deliberately run the battery from full charge down to low, in order to get a better idea of the full charge point and runtime capacity.                                                                                                        |
| OFF     | Offline. The UPS is not providing power to the connected equipment load, for whatever reason.                                                                                                                                                                                      |
| OVER    | Overloaded. Connected equipment load is drawing more current (amps) than the UPS is designed to provide. A small and brief overload may be tolerable. A prolonged or large overload typically results in the UPS output being turned off.                                          |
| TRIM    | Trimming Voltage. Input voltage is somewhat higher than nominal; the UPS is correcting for it. Not all UPSes can do this; those that do not can only switch to battery.                                                                                                            |
| BOOST   | Boosting Voltage. Input voltage is somewhat lower than nominal (brownout); the UPS is correcting for it. Not all UPSes can do this; those that do not can only switch to battery.                                                                                                  |
| FSD     | Forced Shutdown. The UPS has turned off output power in response to softgware command (typically from NUT).                                                                                                                                                                        |
| NOCOMM  | No Communications. NUT is not receiving status information from the UPS. This may mean the UPS signal cable is disconnected, the wrong UPS parameters are configured, the UPS is faulty, a network problem (for SNMP), etc.                                                        |
| WAIT    | Waiting. This has appeared when the daemons were just started; I presume it means the communications channels are still being initialized, so status is not available yet.                                                                                                         |

[UPSLOG Manual](https://networkupstools.org/docs/man/upslog.html)

[scripts/logrotate/nutlogd](https://github.com/networkupstools/nut/blob/master/scripts/logrotate/nutlogd)

- [nut-logger.service](etc/systemd/system/nut-logger.service)
- [upslog.conf](etc/nut.d/upslog.conf)
- [upslog.rc](etc/nut.d/upslog.rc)
- [nut](etc/logrotate.d/nut)

## Links and References

[Understanding the Network UPS Tools (NUT)](https://www.dragonhawk.org/tech/nut/)

[UPSLOG](https://networkupstools.org/docs/man/upslog.html)

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

[ANEEL Resolution 505/2001: Quality Improvement](https://www.cgti.org.br/publicacoes/wp-content/uploads/2016/01/Resoluc%CC%A7a%CC%83o-505_2001-ANEEL-Melhoria-da-Qualidade.pdf)
