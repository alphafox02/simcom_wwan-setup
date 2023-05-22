# Setup the Waveshare SIM7600G

This guide is heavily adapted from the guide published on the waveshare site, available [here](https://www.waveshare.com/wiki/SIM7600G-H_4G_for_Jetson_Nano).

Notes: There seem to be numerous errors and omissions in the instructions as provided by Waveshare. This guide documents my process so that others may avoid many wasted hours of effort. It turns out that you do need `udhcpc` even if you already have `dhclient` installed (dhclient caused issues for me). You also don't need `minicom` or `screen`. There is a way to send and view serial using two terminal windows and built in commands `cat` and `echo`.

## Assumptions

* Your host system is either Linux or OSX
* High degree of comfort with the commandline (i.e. compiling from source)

## Requirements

* Target PC
* Waveshare SIM7600A-H Hat
* Activated SIM with talk/text/data (Mint and T-Mobile are both tested and work)
* High-speed internet connection

## Hardware Setup

1. Power off target
2. Install your activated SIM card in the holder on the underside of the SIM7600A-H hat
3. Install the SIM7600A-H by attaching it via USB to the target
4. Power on the target

* The `PWR` indicator should come on.
* After a moment, the `NET` light should start blinking. 

5. Log into the target over `ssh` and complete the rest of the steps.

## Software Setup

1. `$ sudo apt-get update`
2. `$ sudo apt-get install udhcpc minicom`


## Testing

### Setting up `minicom`

NOTE: It's also possible (and possibly easier) to use `screen`. If you don't have time to deal with this, skip to the "Pure bash shell" instructions at the end of this section.

At this point, the instructions provided by Waveshare call for using `minicom`, but don't provide any hint that it needs to be setup. Instructions for setup can be found [here](https://wiki.emacinc.com/wiki/Getting_Started_With_Minicom) and are summarized below.

1. `$ sudo minicom -s` will greet you with a `configuration` menu

```
            +-----[configuration]------+
            | Filenames and paths      |
            | File transfer protocols  |
            | Serial port setup        |
            | Modem and dialing        |
            | Screen and keyboard      |
            | Save setup as dfl        |
            | Save setup as..          |
            | Exit                     |
            | Exit from Minicom        |
            +--------------------------+
```

1. Arrow down to `Modem and Dialing` and press `enter`
2. Remove "Dialing prefix", "Dialing suffix", and "Hang-up string" entries to match:

```
 +--------------------[Modem and dialing parameter setup]---------------------+
 |                                                                            |
 | A - Init string .........                                                  |
 | B - Reset string ........                                                  |
 | C - Dialing prefix #1....                                                  |
 | D - Dialing suffix #1....                                                  |
 | E - Dialing prefix #2....                                                  |
 | F - Dialing suffix #2....                                                  |
 | G - Dialing prefix #3....                                                  |
 | H - Dialing suffix #3....                                                  |
 | I - Connect string ...... CONNECT                                          |
 | J - No connect strings .. NO CARRIER            BUSY                       |
 |                           NO DIALTONE           VOICE                      |
 | K - Hang-up string ......                                                  |
 | L - Dial cancel string .. ^M                                               |
 |                                                                            |
 | M - Dial time ........... 45      Q - Auto bps detect ..... No             |
 | N - Delay before redial . 2       R - Modem has DCD line .. Yes            |
 | O - Number of tries ..... 10      S - Status line shows ... DTE speed      |
 | P - DTR drop time (0=no). 1       T - Multi-line untag .... No             |
 |                                                                            |
 | Change which setting?     Return or Esc to exit. Edit A+B to get defaults. |
 +----------------------------------------------------------------------------+
```
 
1. Escape to the `configuration` menu
2. Select `Screen and keyboard` and press `enter`.
3. Press `q` to toggle `Local echo` to `Yes`
4. Escape to the `configuration` menu
5. Select `Save setup as dfl` and press `enter`
6. Select `Exit from Minicom` and press `enter`

### On To Testing

For a full list of commands, see the [AT Command Manual](https://www.waveshare.com/w/upload/5/54/SIM7500_SIM7600_Series_AT_Command_Manual_V1.08.pdf).

#### With `minicom`

1. `$ sudo minicom -D /dev/ttyUSB2`
2 Enter `ATI`
3 If you can't see your local echo, you may need to enable it:
	1. Press `ctrl+a` then `z` to bring up the options menu.
	2 Press `e` to enable echo
	3 `esc` to return to the console  

```
ATI

Manufacturer: SIMCOM INCORPORATED
Model: SIMCOM_SIM7600G-H
Revision: SIM7600M22_V2.0
IMEI: 868822040061788
+GCAP: +CGSM

OK
```

#### Pure bash shell

1. `ssh` into the target
2. Start listening to the SIM7600AH serial device: `$ cat < /dev/ttyUSB2`
3. Open a second terminal window and `ssh` into the target computer and complete the following steps.
4. Switch to root user: `$ sudo su`
5. Send a request for product identification info: `# echo -e 'ATI\r' > /dev/ttyUSB2`
6. Now check the first terminal window for the output.

## 4G connection

## Setup Network Interface `wwan0`

1. Check if the `wwan0` interface is present: `$ ifconfig wwan0`
2. Enable the `wwan0` interface: `$ sudo ifconfig wwan0 up`
3. Switch to root user: `$ sudo su`
4. Define network mode as automatic: `# echo -e 'AT+CNMP=2\r' > /dev/ttyUSB2`
5. Connect the NIC to the network: `# echo -e 'AT$QCRMCALL=1,1\r' > /dev/ttyUSB2`
6. Allocate IP: `$ sudo udhcpc -i wwan0`

Now you can use 4G network!

## Installing as `systemd` service

There are scripts included in this repo that allow you to install 4G connectivity at boot using `systemd` service files, a preup script and a poststop script to automate the steps in the "Setup Network Interface `wwan0`" section above.

It's recommended that you clone the repo locally on the target computer.

1. `$ git clone https://github.com/alphafox02/simcom_wwan-setup.git`
2. `$ cd simcom_wwan-setup`
3. `$ chmod +x install.sh uninstall.sh update.sh`
4. To install: `$ sudo ./install.sh`
5. To uninstall: `$ sudo ./uninstall.sh` 
6. To update: `$ git pull; sudo ./update.sh`

* This service is disabled by default and will not start at boot.
* To enable, run `$ sudo systemctl enable simcom_wwan@wwan0.service`
* To disable, run `$ sudo systemctl disable simcom_wwan@wwan0.service`
* To start the service and 4G LTE connectivity: `$ sudo systemctl start simcom_wwan@wwan0.service`
* To stop the service and 4G LTE connectivity: `$ sudo systemctl stop simcom_wwan@wwan0.service`
* To check the status of the service: `$ sudo systemctl status simcom_wwan@wwan0.service`

### To make the `sim_comwwan@wwan0` service wait for the USB device:

1. `$ sudo nano /etc/udev/rules.d/99-usb-4g.rules`
2. Add the line: `SUBSYSTEM=="tty", KERNEL=="ttyUSB2", TAG+="systemd", ENV{SYSTEMD_WANTS}+="simcom_wwan@wwan0.service"`
3. Ctrl-X, Y, Enter (Save and close)

### Test the changes

1. `$ sudo reboot`

### References to consider
* https://techship.com/faq/how-to-integrate-simcom-sim7500-sim7600-series-linux-ndis-driver-without-rebuilding-kernel/
* https://techship.com/faq/how-to-automated-connection-establishment-with-sim7600-series-modules-when-using-rndis-usb-mode/


