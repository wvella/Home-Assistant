# These are my notes for setting up *Home Assistant* on my Synology NAS in Docker from scratch.

## Why am I documenting this?

I've started to play around with Z-Wave and Smart Home recently, and I am learning a bucket load.  I have a strong technical background, and work in Integration and APIs in my day job, but I hope to make setting up a Smart Home accessible to as many people as possible.

There is a wealth of knowledge out there and the community is amazing. The least I could do is contribute back by sharing my experiences all the way from the start.

I'm keen for feedback so please feel free to fork this, contribute and share experiences!

## Installation

Here we go! Assuming you have your Synology NAS up and running, connect to the NAS via ssh;

`ssh wvella@10.0.10.100`

Once logged in, start a new container in Docker;

`sudo docker run --restart always -d --name="nas-south-homeassistant1" -v /volume3/docker/homeassistant/config:/config --device=/dev/zwave -e TZ=Australia/Melbourne --net=host homeassistant/home-assistant:stable`

The parameters mean;

- `sudo` Run the command in privledged mode
- `docker run` Run a command in a new container
-  `--restart` Always restart after the NAS starts up
- `-d` Run container in background and print container ID
- `--name` Friendly name for the container
- `-v` Mount a volume in the container to a folder on the NAS
- `--device` Mount the Z-Wave USB stick in the container
- `-e` Set the TimeZone to you local time sonze
- `--net` Set the network of the container to the same network as the Docker host. In this case, our Synology NAS.


>**NOTE**: Make sure the local folder on the NAS is created to store the configuration files. This folder will be mapped in the container so the configuration files are persisted after a restart..

Once the container has been created, you can view if it's running;

`sudo docker ps`

To connect into the container and open up a `bash` shell;

`sudo docker exec -it <container name> /bin/bash`

To view the Home Assistant web page, access the following link. 
[Home Assistant - Lovelace UI](http://10.0.10.100:8123)

For basic configuration (great starting point), refer to the following link.
[Configuring Home Assistant](https://www.home-assistant.io/docs/configuration/)

## Post Installation

### Mounting the Z-Wave USB Stick (with a persistence mapping)

By default, the Z-Wave USB stick will show up as either /dev/ttyACM0 or /dev/ttyACM1 and can change each time the NAS is restarted, or the USB stick is removed and plugged back in. To persist the USB stick to an 'alias' name, we can use `udev` as below;

1. Using `cat /proc/bus/usb/device` find the *Vendor* and *Product* ID of the AEOTEC Z-Wave USB Stick. It will largely be the same across the Gen 5 fleet of USB sticks.
2. In `/lib/udev/rules.d` create a new file, such as `50-usb-zwave.rules`. 
3. Add the following content;
`SUBSYSTEM=="tty", ATTRS{idVendor}=="0658", ATTRS{idProduct}=="0200", SYMLINK+="zwave"`
> NOTE: I spent a considerable amount of time troubleshooting the udev file above and couldn't work out why it wasn't working. It ended up being a copy-and-paste error and the way the quotes were copied.
4. Run `udevadm control --reload-rules && udevadm trigger` to reload the udev files (no need to restart the NAS.
5. Create a schedule task when the NAS starts by creating the following file: `/usr/local/etc/rc.d/openhab-zwave-usbpermissions.sh`
6. Add the following content;
`#!/bin/sh
chown -R openhab:openhab /dev/ttyACM0
exit 0`

### Install the Z-Wave Integration

Once Home Assistant has been setup, we need to install the Z-Wave integration.  Under *Configuration* -> *Integrations*, Install the **Z-Wave** Integration. 

Ensure the *USB Path* is configured as `/dev/zwave`. (This is a custom mapping that is auto-mapping the USB drive. Need to add more details here.

Detailed instructions can also be found [here](https://www.home-assistant.io/docs/z-wave/installation/).

Once Home Assistant is installed, restart the container so the Z-Wave stick can be detected. When Home Assistant starts-up, you should now see the following;

1. *Z-Wave Network Started* in the Z-Wave Network Management page. This can be found in Configuration -> Z-Wave.
2. In the Z-Wave Network Management page, the Node should now appear in the Z-Wave Node Management list. In my case, this appears as `AEON Labs ZW090 Z-Stick Gen5 AU (Node:1 Complete).

### Initial Reading

It's always a good idea to read the getting-started guides to familiarse yourself with the basics. Some good pages are;

[YAML](https://www.home-assistant.io/docs/configuration/yaml/)
[Setup basic information](https://www.home-assistant.io/docs/configuration/basic/)
[Adding devices to Home Assistant](https://www.home-assistant.io/docs/configuration/devices/)
[Customizing entities](https://www.home-assistant.io/docs/configuration/customizing-devices/)
[Troubleshooting your configuration] (https://www.home-assistant.io/docs/configuration/troubleshooting/)
[Securing](https://www.home-assistant.io/docs/configuration/securing/)

A good note;

The only characters valid in entity names are:

* Lowercase letters
* Numbers
* Underscores

If you create an entity with other characters then Home Assistant may not generate an error for that entity. However you will find that attempts to use that entity will generate errors (or possibly fail silently).

Wow - something I just found! A live chat site to help people with troubleshooting issues. It can be found here [Discord Chat](https://discord.gg/c5DvZ4e)

#### Z-Wave Graph

Out of the box, Home Assistant doesn't contain an Z-Wave mesh graph which is super handy when troubleshooting connecitvity issues. Here is an amazing Panel to visual the Z-Wave Mesh. 

[Z Wave Graph for Home Assistant](https://gist.github.com/AdamNaj/cbf4d792a22f443fe9d354e4dca4de00)

Some files will need to be added to the Home Assistant configuration, so to connect *into* the Container, use the following;

* Find the container ID with `sudo docker ps`
* Using the container ID, run the following `sudo docker exec -it 2bf60533ffa8 /bin/bash`

### Z-Wave Devices - Adding and Removing

Go to the following [page](https://www.home-assistant.io/docs/z-wave/adding/) to learn how to add and remove devices. One the devices are added, it's time to reanme them to something nicer. One nice feature in adding the Z-Wave device is that it can be done *in-place* without having to remove the USB stick. Just start the *Add node* option in Home Assistant and click the button on the device. If the Mesh is working, the device should be able to be added in. Always remember to also *Heal* the network so the Mesh can rebuild and optimise the path.

> **A WORD OF WARNING**: Z-Wave works on a mesh network, that is not line of sight (unlike Wifi). This means, the more Z-Wave devices that are added, the stronger the network becomes. On the converse, having only a few Z-Wave devices will cause a lot of dead nodes, which I something I wrestled with - until I figured this out.

### Renaming the Devices

Out of the box, Home Assistant will give the Node, the Device & the Entity a default name. I like to give my devices a more friendly name, so here is how I have renamed my assets.

#### Node Names

In the Z-Wave Node Management page, select the node you want to rename and select `Node Information`. Rename this node (by clicking on the Gear) to your own name. Remember, this is naming the *physical* Z-Wave node, not the devices the node exposes (some switches expose 2 devices, such as the AEOTEC Dual Nano Switch).

I like to give my Nodes the following naming pattern;

`Name Override`: [Node ID] - [Area] - [Location] - [Device Type]. For example; `Node1 - Backyard - Side Pathway - Dual Nano Switch`
`Entity ID`: [domain].[location] _ [devicetype]. For example; `zwave.backyard_sidepathway_dualnanoswitch`

>**NOTE**: Note the lowercase and no spaces in the Entity ID. The domain is also fixed.

#### Device Names

In *Configuration* -> *Devices* find the device you want to rename. Rename this device (by clicking on the Gear) to your own name. 

>**NOTE**: Most of my devices are Z-Wave Dual Nano Switches which are seen as 'two seperate switches'. Intersestingly though, in this Devices page, a total of three (3) devices appear. I can only assume one device is the 'master switch' and the other two devices, each individual switch.

I like to give my Devices the following naming pattern;

`Name Override`: [Friendly Device Name]. For example; `Pathway Lights`
`Area`: [Area of device]

#### Entitiy Names

I like to give my Entites the following naming pattern;

`Name Override`: [[Device Type] [Sensor]]. For example; `Dual Nano Switch Heat`
`Location`: [Location of device]
`Entity ID`: [domain].[location]_[devicetype]_[switchtype]_[Sensor]. For example; `sensor.sidepathway_dualnanoswitch_lights_heat`

>**NOTE**: Note the lowercase and no spaces in the Entity ID. Locations are defined in the Area Registry. The domain is also fixed.

>**NOTE**: I also like to remove devices that are connected, but not used. This is the case on Dual Nano Switches where I am only using 1 of the 2 available switches.

### Cleaning up old Devices and Entities

After a few attempts in trying to get the Z-Wave devices added, I ended up with a whole bunch of devices and entities hanging around in the database which caused clutter. These can be cleaned up by editing a few JSON files. The process I followed is;

1. Open the `Home Assistant\config\.storage\core.device_registry` file in a good file editor (like Sublime Text or Visual Studio code).
2. Open the `Home Assistant\config\.storage\core.entity_registry` file in a good file editor (like Sublime Text or Visual Studio code).

> **Note**: Files in the .storage directory. If you want to see hidden files in macOS press `Command + Shift + .` and the files will appear. Repeat this sequence to hide the hidden files.

> **Note**: ALWAYS create a backup of these files before you edit them. One typo may invalidate the file.

>**Note**: For some reason, sometimes I don't have access to edit these files. To workaround this issue, I granted full access to the file with the following (via a SSH session and elevated to sudo);

> `chmod 777 core.device_registry`
> `chmod 777 core.entity_registry`
> 
> Once you are done editing the file, revert the permissions and access back with the following;
> 
> `chmod 644 core.device_registry`
> `chmod 644 core.entity_registry` 
> `chown root core.device_registry`
> `chown root core.entity_registry` 
> `chgrp root core.device_registry`
> `chgrp root core.entity_registry` 

-rw-r--r--  1 root   root

3. In the `core.device_registry` file, find the device you want to delete. An example is below;

`{
                "area_id": null,
                "config_entries": [
                    "06b2a1a9304b40f6b8c005864ecd63b6"
                ],
                "connections": [],
                "id": "25f886b651d04a15ac1b2781f6461d8b",
                "identifiers": [
                    [
                        "zwave",
                        2
                    ]
                ],
                "manufacturer": "AEON Labs",
                "model": "ZW096 Smart Switch 6",
                "name": "AEON Labs ZW096 Smart Switch 6",
                "name_by_user": null,
                "sw_version": null,
                "via_device_id": "a6e6c7e0b4274ddca5357015eb1265ec"
            },`
						
4. The whole entry can be removed. Remember to preserve the structure of the JSON file.
5. Copy the value from `"id"` and find this id in the `core.entity_registry` file. There may be multiple entity entries. These can also be removed.
6. Save both files and restart Home Assistant.

## Creating device groups

## Turning off the Garage Alarm Sound

The default behavior of the *Aeotec Garage Door Controller* is to ring a loud alarm for 5 seconds before the garage door is opened and closed. First thing I wanted to do is turn this off!

I found this [great guide](https://aeotec.freshdesk.com/support/solutions/articles/6000131922-disable-alarm-sound-in-the-garage-door-controller) but in summary;

1. Change Parameter 32 (Startup ringtone) to 0
2. Change Parameter 37 (Configure alarm mode for when the door is opening) to 167837952
3. Change Parameter 38 (Configure the alarm mode when the garage door is closing) to 83951872
4. Change Parameter 39 (Configure alarm mode when the garage door is in unknown state) to 117506304
5. Change Parameter 40 (Configure the alarm mode when the garage door is in closed position) to 16843008