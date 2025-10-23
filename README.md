# heos2mqtt

## Setup

Copy `heos2mqtt-events` and `heos2mqtt-commands` into `/usr/local/bin`.

Copy `heos2mqtt-events@.service` and `heos2mqtt-commands@.service` to `/etc/systemd/system`.

Install `netcat` and `mosquitto_{pub,sub}` (example for debian - your particular OS might have a different package manager and different package names)

```bash
sudo apt install netcat-traditional mosquitto-clients
```

Create an environment file at `/etc/default/heos2mqtt`:

```bash
cat << EOF | sudo tee /etc/default/heos2mqtt
MQTT_IP=192.168.1.171
MQTT_USER=heos
MQTT_PASS=mysupersecretpassword
EOF
```

**Note**: since this contains a password, you might want to restrict access to only the root account:

```bash
sudo chmod 400 /etc/default/heos2mqtt
cat /etc/default/heos2mqtt
# cat: /etc/default/heos2mqtt: Permission denied
```

Enable the services, passing the IP of your HEOS device as the instantiated parameter:

```bash
sudo systemctl enable --now heos2mqtt-events@192.168.1.219
sudo systemctl enable --now heos2mqtt-commands@192.168.1.219
```

Check the logs:

```bash
journalctl -f -xeu heos2mqtt-events@192.168.1.219
# Oct 23 13:13:17 pi-zero heos2mqtt-events[3189]: HEOS connection: 192.168.1.219:1255
# Oct 23 13:13:17 pi-zero heos2mqtt-events[3189]: MQTT connection: 192.168.1.171:1883
# Oct 23 13:13:17 pi-zero heos2mqtt-events[3189]: MQTT topic: heos/raw/events
# Oct 23 13:13:17 pi-zero heos2mqtt-events[3195]: {"heos": {"command": "system/register_for_change_events", "result": "success", "message": "enable=on"}}
# Oct 23 13:13:21 pi-zero heos2mqtt-events[3195]: {"heos": {"command": "event/player_now_playing_progress", "message": "pid=-1163911007&cur_pos=379000&duration=0"}}
```

(my speaker is currently playing, so we get '`player_now_playing_progress`' events)

```bash
journalctl -f -xeu heos2mqtt-commands@192.168.1.219
# Oct 23 13:28:11 pi-zero heos2mqtt-commands[3705]: HEOS connection: 192.168.1.219:1255
# Oct 23 13:28:11 pi-zero heos2mqtt-commands[3705]: MQTT connection: 192.168.1.171:1883
# Oct 23 13:28:11 pi-zero heos2mqtt-commands[3705]: MQTT topic: heos/raw/commands
```

(no commands yet - let's go do that now)

## Sending a command

You can see the list of valid commands at [HEOS_CLI_ProtocolSpecification-Version-1.17.pdf](https://rn.dmglobal.com/usmodel/HEOS_CLI_ProtocolSpecification-Version-1.17.pdf).

Use your favourite MQTT client to send a `heos://player/get_players` command. In this example I'll just use `mosquitto_pub` which I've just installed on this machine, but you can do this from whatever interface you have available, as long as it can access the MQTT broker:

```bash
mosquitto_pub -h 192.168.1.171 -p 1883 -u heos --pw "$MY_MQTT_PASS" --topic heos/raw/commands -m 'heos://player/get_players'
```

Now let's go back and check the logs:

```bash
journalctl -xeu heos2mqtt-commands@192.168.1.219
# Oct 23 13:42:41 pi-zero heos2mqtt-commands[3981]: HEOS connection: 192.168.1.219:1255
# Oct 23 13:42:41 pi-zero heos2mqtt-commands[3981]: MQTT connection: 192.168.1.171:1883
# Oct 23 13:42:41 pi-zero heos2mqtt-commands[3981]: MQTT request topic: heos/raw/commands
# Oct 23 13:42:41 pi-zero heos2mqtt-commands[3981]: MQTT response topic: heos/raw/commands/response
# Oct 23 13:42:45 pi-zero heos2mqtt-commands[3987]: heos://player/get_players
# Oct 23 13:42:45 pi-zero heos2mqtt-commands[3989]: {"heos": {"command": "player/get_players", "result": "success", "message": ""}, "payload": [{"name": "Home Theater", "pid": 1816109296, "model": "Denon AVR-X2700H", "version": "3.88.350", "ip": "192.168.1.136", "network": "wifi", "lineout": 0, "serial": "****"}, {"name": "Kitchen", "pid": -1163911007, "model": "Denon Home 150", "version": "3.88.350", "ip": "192.168.1.219", "network": "wifi", "lineout": 0, "serial": "****"}]}
```

If we were also listening to the response channel, we'd also see it transmitted back:

```bash
mosquitto_sub -h 192.168.1.171 -p 1883 -u heos --pw "$MY_MQTT_PASS" --topic heos/raw/commands/response
# {"heos": {"command": "player/get_players", "result": "success", "message": ""}, "payload": [{"name": "Home Theater", "pid": 1816109296, "model": "Denon AVR-X2700H", "version": "3.88.350", "ip": "192.168.1.136", "network": "wifi", "lineout": 0, "serial": "****"}, {"name": "Kitchen", "pid": -1163911007, "model": "Denon Home 150", "version": "3.88.350", "ip": "192.168.1.219", "network": "wifi", "lineout": 0, "serial": "****"}]}
```
