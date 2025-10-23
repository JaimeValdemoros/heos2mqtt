# heos2mqtt

Copy `heos2mqtt-events` into `/usr/local/bin`.

Copy `heos2mqtt-events@.service` to `/etc/systemd/system`.

Install `netcat` and `mosquitto`:

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

Enable the service, passing the IP of your HEOS device as the instantiated parameter:

```bash
sudo systemctl enable --now heos2mqtt-events@192.168.1.219
```

Follow the logs:

```bash
journalctl -f -xeu heos2mqtt-events@192.168.1.219
```