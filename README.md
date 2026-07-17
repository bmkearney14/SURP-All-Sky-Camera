# SURP-All-Sky-Camera
Code documentation for the SURP 2026 All-Sky camera Rapsberry Pi project.


# AllSky Networking Setup

Here are instructions for how to setup static IPs, the firewall, and simultaenous WiFi/Ethernet connection for the Smith Roof All Sky Raspberry Pi.

### Static IP

First, we will set a static IP for the Pi. Run the following command:

```
sudo nmtui
```

This will open the network manager text user interface on the Pi. Select "Wired connection 1", and then change the IPv4 Configuration from "Automatic" to "Manual".

Under IPv4 Configuration, select "Show" to edit the configuration, then change the following:

```
Addresses: 192.168.65.150
Gateway: 192.168.65.1
DNS servers: 192.168.65.1
```

Don't change anything else. Go to the bottom of the page and hit "OK" to save your changes.

### Set WiFi as default network interface

By default, the Pi will set both the WiFi and Ethernet adapters as the default network interface. So if both are connected, it will try to send all internet traffic through the ethernet adapter. We don't want that, as the only connections we can make over the ethernet are to other devices on the local network. To fix this, first type the following command:

```
ip route
```

If the network is not set up properly, you will see two network interfaces with "default" next to their names.

You can use the following command to get the names of network interfaces being used on the Pi.

```
nmcli connection show
```

This will use the network manager command line interface tool on the Pi to show you active connections. If you have both the ethernet cable plugged in and WiFi connected, you should see both devices `wlan0` and `eth0`.

If `nmcli` isn't working, you can try running `systemctl status NetworkManager` to figure out what the issue is.

Once you verify both network adapters are working correctly, use the following command to make the Pi not use ethernet as the default network interface. Replace `<eth0-connection-name>` with the name of the network interface you got from `nmcli connection show`. It should be `"Wired connection 1"`. Use `""` to encapsulate names that contain spaces in command line arguments.

```
nmcli connection modify <eth0-connection-name> ipv4.never-default true
```

Next we will change network metrics to ensure WiFi is preferred over ethernet. Run the following commands, and again swap out `<eth0-connection-name>` and `<wlan0-connection-name>` for the real network interface names. The WiFi interface name should be `Registered4OSU`. Note that a lower number means higher priority with network metrics.

```
nmcli connection modify <wlan0-connection-name> ipv4.route-metric 100
nmcli connection modify <eth0-connection-name> ipv4.route-metric 200
```

Finally, run these commands to apply the settings.

```
sudo nmcli connection up <eth0-connection-name>
sudo nmcli connection up <wlan0-connection-name>
```

You should now be able to use the internet over WiFi and connect to devices through ethernet LAN at the same time.

### Setup MQTT for indi-allsky

First, run this command to enable `mosquitto` on the Pi:

```
sudo systemctl enable mosquitto
```

To start, generate the `mosquitto` config file using the `setup_mosquitto_mqtt.sh` script under `misc/` in the `indi-allsky` repository. Once you are in the correct folder, run the following command.

```
./setup_mosquitto_mqtt.sh
```

Within the script, set the username to `indi-allsky` and make a secure password. You will need it later.

Next, we will setup the firewall. Run the following command to install `ufw`.

```
sudo apt install ufw -y
```

Enable the firewall.

```
sudo ufw enable
```

Open the port used to communicate with MQTT.

```
sudo ufw allow 1883
```

Reload the firewall configuration.

```
sudo ufw reload
```

Now we will setup MQTT in `indi-allsky`. Open the web interface and login as the surp/admin user. Next, navigate to the Config page. It will be under one of the panels on the left. Click on the "MQTT" link at the top. On this page, enable the following settings:

```
Enable MQTT Publishing: (ENABLED)
MQTT Host: 192.168.65.150
MQTT Transport: tcp
MQTT Protocol: v5.0
Port: 1883
Username: indi-allsky
Password: <password you set in the setup script>
MQTT Base Topic: indi-allsky
MQTT QOS: 0
Use TLS (DISABLED)
Disable Certificate Validation (ENABLED)

Enable Image Publishing (ENABLED)
```

Check reload on save, then click the save button. Some of these settings won't be applied until you restart the Pi, so run a `sudo reboot now` to restart the Pi before validating the settings change worked. MQTT publishing should now be working.

Tip: You can navigate to the folder `/var/log/indi-allsky` and use the command `cat indi-allsky.log | grep ERROR` to see what errors are getting logged if anything is not working correctly.
