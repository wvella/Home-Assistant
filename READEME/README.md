# These are my notes for setting up *Home Assistant* on my Synology NAS in Docker from scratch.

## Installation

Firstly, connect to the NAS via ssh;

`ssh wvella@10.0.10.100`

Once logged in, start a new container in Docker;

`sudo docker run --restart always -d --name="nas-south-homeassistant1" -v /volume3/docker/homeassistant/config:/config --device=/dev/zwave -e TZ=Australia/Melbourne --net=host homeassistant/home-assistant:stable`

Note: Make sure you have configured a local folder on the NAS for the configuration files. This will be mapped in the container so the configuration files are persisted.

Once the container has been created, you can view if it's running;

`sudo docker ps`

To view the Home Assistant web page, access the following link. 
[Home Assistant - Lovelace UI](http://10.0.10.100:8123)

For basic configuration (great starting point), refer to the following link.
[Configuring Home Assistant](https://www.home-assistant.io/docs/configuration/)

## Post Installation

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

A word of warning. Z-Wave works on a mesh network, that is not line of sight (unlike Wifi). This means, the more Z-Wave devices that are added, the stronger the network becomes. On the converse, having only a few Z-Wave devices will cause a lot of dead nodes, which I something I wrestled with - until I figured this out.

### Renaming the Devices

After a few attempts in trying to get the Z-Wave devices added, I ended up with a whole bunch of devices and entities hanging around in the database which caused clutter. These can be cleaned up by editing a few JSON files. The process I followed is;

1. Open the `Home Assistant\configs\.storage\core.device_registry` file in a good file editor (like Sublime Text or Visual Studio code).
2. Open the `Home Assistant\configs\.storage\core.entity_registry` file in a good file editor (like Sublime Text or Visual Studio code).

Note: Files in the .storage directory. If you want to see hidden files in macOS press `Command + Shift + .` and the files will appear. Repeat this sequence to hide the hidden files.
Note: ALWAYS create a backup of these files before you edit them. One typo may invalidate the file.

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
5. Copy the value from `"via_device_id"` and find this id in the `core.entity_registry` file. There may be multiple entity entries. These can also be removed.
6. Save both files and restart Home Assistant.