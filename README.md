# IDM Scanner

#### Disclaimer: Usage of this scripts happens at your own risk!

## Don’t do anything that is not mentioned in the tutorial, but be sure to do what is mentioned.

* [1. Mount IDM](#1-mount-idm)
* [2. Install IDM Klipper Module](#2-install-idm-klipper-module)
* [3. Configure Klipper for IDM](#3-configure-klipper-for-idm)
  - [3.1 Serial Query](#31-serial-query)
  - [3.2 Canbus Query](#32-Canbus-query)
* [4. Calibrate IDM](#4-calibrate-idm)
* [5. Basic First Tests](#5-basic-first-tests)
* [6. Calibrate Bed Mesh](#6-calibrate-bed-mesh)
* [7. Print](#7-print)
* [8. Enter High-Speed Mode](#8-enter-high-speed-mode)
* [9. Enable The Accelerometer](#9-enable-the-accelerometer)
* [10. Supported Commands](#10-supported-commands)
* [11. Update IDM](#11-update-idm)
  - [11.1 Update Canboot](#111-update-canboot)
  - [11.2 Update Firmware](#112-update-firmware)

## 1. Mount IDM
Mount the IDM-Scanner to your 3D printer toolhead, nominally 2.6mm recessed from the nozzle in Z.

To ensure accuracy, please install so that the top surface of the sensor coil plate is lower than the bottom surface of the heating block as much as possible.

[Voron Mount](/STLs/Voron)

## 2. Install IDM Klipper Module
Clone ```IDM``` from git and run the install script:
```
cd ~
git clone https://github.com/ModularPrintingSystem/IDM.git
./IDM/install.sh
```

You can advance to the next sections and edit your config files while you wait.

## 3. Configure Klipper for IDM
Add the idm configuration to your printer config:
```
[idm]
serial:
#   Path to the serial port for the idm device. Typically has the form
#   /dev/serial/by-id/usb-idm_idm_...
#canbus_uuid:
speed: 40.
#   Z probing dive speed.
lift_speed: 5.
#   Z probing lift speed.
backlash_comp: 0.5
#   Backlash compensation distance for removing Z backlash before measuring
#   the sensor response.
x_offset: 0.
#   X offset of idm from the nozzle.
y_offset: 21.1
#   Y offset of idm from the nozzle.
trigger_distance: 2.
#   idm trigger distance for homing.
trigger_dive_threshold: 1.5
#   Threshold for range vs dive mode probing. Beyond `trigger_distance +
#   trigger_dive_threshold` a dive will be used.
trigger_hysteresis: 0.006
#   Hysteresis on trigger threshold for untriggering, as a percentage of the
#   trigger threshold.
cal_nozzle_z: 0.1
#   Expected nozzle offset after completing manual Z offset calibration.
cal_floor: 0.1
#   Minimum z bound on sensor response measurement.
cal_ceil:5.
#   Maximum z bound on sensor response measurement.
cal_speed: 1.0
#   Speed while measuring response curve.
cal_move_speed: 10.
#   Speed while moving to position for response curve measurement.
default_model_name: default
#   Name of default idm model to load.
mesh_main_direction: x
#   Primary travel direction during mesh measurement.
#mesh_overscan: -1
#   Distance to use for direction changes at mesh line ends. Omit this setting
#   and a default will be calculated from line spacing and available travel.
mesh_cluster_size: 1
#   Radius of mesh grid point clusters.
mesh_runs: 2
#   Number of passes to make during mesh scan.
```
Please pay attention to the x-y offset in the adjustment configuration. Make sure that during the calibration process, the nozzle will move the coil to the x-y position of the original nozzle.

Remember to set ```[bed_mesh]``` otherwise an error will be reported.

Delete the ```[probe]``` module in your configuration and modify the z limit:
```
[stepper_z]
endstop_pin: probe:z_virtual_endstop # use idm as virtual endstop
homing_retract_dist: 0 # idm needs this to be set to 0
```

If you have already configured ```[safe_z_home]``` or ```[homing_override]```, you can ignore following step:

Set the safe x-y position for z homing (typically the bed center):
```
[safe_z_home]
home_xy_position: <your x-axis center>, <your y-axis center>
z_hop: 10
```

### 3.1 Serial Query
The serial query command is:
```
ls /dev/serial/by-id/*
```

### 3.2 Canbus Query
Clone ```Katapult``` from git:
```
cd ~
git clone https://github.com/Arksine/katapult.git
```

To enable  the canbus on the raspberry pi you need to create the interface file:
```
sudo nano /etc/network/interfaces.d/can0
```

And paste the following content:
```
allow-hotplug can0
iface can0 can static
 bitrate 1000000
 up ifconfig $IFACE txqueuelen 1024
```

To bring the canbus interface ```can0``` up simply run:
```
sudo ifup can0
```

The canbus query command is:
```
python3 ~/katapult/scripts/flash_can.py -q
```

## 4. Calibrate IDM
Home the machine in X and Y:
```
G28 X Y
```

Position the nozzle in the centre of the bed. You will nee dto adjust the coordinates for your machine, or feel free to use the web interface
```
G0 <your x-axis center>, <your y-axis center>
```

Start the calibration process:
```
IDM_CALIBRATE
```

Proceed through a standard nozzle paper offset test until the paper drags. Remove the paper and accept the position:
```
ACCEPT
```

The sensor response will be automatically measured and fit to a model. Save the results to your config file:
```
SAVE_CONFIG
```

## 5. Basic First Tests
You can take a spot measurement with IDM:
```
IDM_QUERY
```

Your Machine will home z with IDM:
```
G28 Z
```

You can test the accuracy:
```
PROBE_ACCURACY
```

You can measure the backlash of your Z axis:
```
IDM_ESTIMATE_BACKLASH
```

## 6. Calibrate Bed Mesh
Run a scan mode mesh:
```
BED_MESH_CALIBRATE
```

## 7. Print
You're now ready to print.

On the first print, you'll want to use babystepping via the GUI to fine adjust the first layer offset.

After the print finishes, the offset can be automatically applied to the model with ```Z_OFFSET_APPLY_PROBE``` command for future prints.

## 8. Enter High-Speed Mode
Lowering the ```horizontal_z_move``` in ```z_tilt``` or ```quad_gantry_level``` to below ```trigger_distance``` + ```trigger_dive_threshold``` (default is 3) can make the leveling enter high-speed mode. If it is too low, you can raise the ```trigger_dive_threshold``` appropriately so that ```horizontal_z_move``` can be raised higher.

## 9. Enable The Accelerometer
For versions that include an accelerometer (lis2dw), you can enable the accelerometer by adding the following to the configuration:
```
[lis2dw]
cs_pin: idm:PA3
spi_bus: spi1

[resonance_tester]
accel_chip: lis2dw
probe_points:
    <your x-axis center>, <your y-axis center>, 20
```

## 10. Supported Commands
Enter ```IDM``` in the console and press the tab key to see all supported commands.

## 11. Update IDM
After IDM is connected to a power source, quickly unplug the power cord and plug it in again. The LED will start to flash slowly, indicating that it has entered canboot. If it is not successful, please repeat this step again.

### 11.1 Update Canboot
Query IDM Canbus UUID:
```
python3 ~/klipper/lib/canboot/flash_can.py -q
```

Execute the following command:
```
cd ~/katapult/scripts python3 flashtool.py -i can0 -f ~/IDM/Canboot/Canboot_1M.bin -u <found uuid>
```
### 11.2 Update Firmware
Query IDM Canbus UUID:
```
python3 ~/klipper/lib/canboot/flash_can.py -q
```

Execute the following command:
```
cd ~/katapult/scripts python3 flashtool.py -i can0 -f ~/IDM/Firmware/IDM_CAN_8kib_offset_1M.bin -u <found uuid>
```