# Nobreak Ts Shara on Ubuntu 20.04 with NUT

[NOBREAK UPS SENOIDAL UNIVERSAL 2200 #4222](https://tsshara.com.br/produto/nobreak-ups-senoidal-universal-2200-2/)

<img alt="Front panel" src="images/tsshara_front_panel.png" width="200" height="200"> <img alt="Rear panel" src="images/tsshara_rear_panel.png" width="200" height="200"> <img alt="Packaging" src="images/tsshara_packaging.png" width="200" height="200">

## Installation

```bash
sudo apt install nut
```

## Configuration

The manufacturer indicates in the specifications that the UPS supports "Intelligent Communication: with USB interface".

After unpacking and turning on the UPS, you can see a new USB device `STMicroelectronics Virtual COM Port`.

In my case with vendor id **0483** and product id **5740**.

```bash
Bus 003 Device 018: ID 0483:5740 STMicroelectronics Virtual COM Port
```

So for the NUT service tobe able to0 use the new serial interface, it is necessary 
to create a rule for the UDEV to change permissions. 
It is also interesting to create a symbolic link so that the interface remains constant.


```bash
SUBSYSTEM=="tty",ATTRS{idVendor}=="0483",ATTRS{idProduct}=="5740",GROUP="nut",OWNER="nut",MODE="0660",SYMLINK+="ttyTSSHARA"
```
