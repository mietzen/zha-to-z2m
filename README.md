## !!! DISCLAIMER !!!

**This worked for me and hopefully will work for you, but MAKE BACKUPS!\
\
If you don't understand any of the code below, DON'T run this!\
\
Also don't do this when you are in a hurry — this might completely destroy your Home Assistant ZHA setup.\
You have been warned... watch out for dragons...**

# Migrate ZHA to Z2M

This script will migrate your [ZHA](https://www.home-assistant.io/integrations/zha/) device to [Zigbee2MQTT](https://www.zigbee2mqtt.io/) using `core.device_registry` and `zigbee.db` from Home Assistant and will generate/edit `devices.yaml`, `configuration.yaml`, and `database.db` in `/homeassistant/zigbee2mqtt`.

It will also install [yq](https://github.com/mikefarah/yq).

Thanks to [seidtgeist](https://github.com/seidtgeist), [toine512](https://github.com/toine512) and [teal-bauer](https://github.com/teal-bauer), whose work this is based on.

Tested on arm64 (amd64 prepared but untested) on Home Assistant Operating System.

On other Home Assistant setups this will only work **with** modifications!

## Before Migrating

For this to work, you need to:
 - Install Zigbee2MQTT
 - Install Mosquitto
 - Stop ZHA
 - Start Zigbee2MQTT
 - Stop Zigbee2MQTT

Afterward, connect to home assistant via `ssh` and run the commands below:

## Start Migration

```shell
set -euo pipefail

cd /homeassistant

# Install yq
wget "https://github.com/mikefarah/yq/releases/latest/download/yq_linux_$(arch | sed -e 's/x86_64/amd64/' -e 's/aarch64/arm64/')" -O /usr/bin/yq; chmod +x /usr/bin/yq

# On my instalation the tables got postfix versions string like _v11 _v12 _v13, we need to get the latest version:
VERSION=$(sqlite3 zigbee.db "SELECT COALESCE('_v' || MAX(CAST(substr(name, instr(name, '_v') + 2) AS INT)), '') FROM sqlite_master WHERE name LIKE '%_v%' AND type='table';")

# Append the Database:
sqlite3 zigbee.db "SELECT CONCAT('{"'"'"id"'"'": ', (ROW_NUMBER() OVER (ORDER BY d.ieee)) + 1, ', "'"'"type"'"'": "'"'"EndDevice"'"'", "'"'"ieeeAddr"'"'": "'"'"0x', REPLACE(d.ieee, ':', ''), '"'"'", "'"'"nwkAddr"'"'": ', d.nwk, '}') FROM devices${VERSION} d JOIN endpoints${VERSION} e ON d.ieee = e.ieee ;" >> /homeassistant/zigbee2mqtt/database.db

# Get the ZHA Zigbee config:
ZIGBEE_CONF=$(sqlite3 zigbee.db "SELECT backup_json FROM network_backups${VERSION} ORDER BY id DESC LIMIT 1 ;" | jq -c)

# Parse and transform:
pan_id=$(echo "$ZIGBEE_CONF" | jq -r '.network_info.pan_id' | awk '{print "0x" $1}')
ext_pan_id=$(echo "$ZIGBEE_CONF" | jq -r '.network_info.extended_pan_id' | awk -F: '{for (i=8; i>0; i--) printf "0x%s%s", $i, (i>1?", ":"")}')
channel=$(echo "$ZIGBEE_CONF" | jq -r '.network_info.channel')
network_key=$(echo "$ZIGBEE_CONF" | jq -r '.network_info.network_key.key' | awk -F: '{for (i=1; i<=NF; i++) printf "0x%s%s", $i, (i<NF?", ":"")}')

# Delete the advanced key from the default configuration
yq -i 'del(.advanced)' /homeassistant/zigbee2mqtt/configuration.yaml

# Append the parsed data:
cat <<EOF >> /homeassistant/zigbee2mqtt/configuration.yaml
advanced:
  pan_id: $pan_id
  ext_pan_id: [$ext_pan_id]
  channel: $channel
  network_key: [$network_key]
devices: devices.yaml
EOF

# Create the friendly device names:
jq -c '.data.devices[]' /homeassistant/.storage/core.device_registry | while read -r dev; do
    id=$(echo "$dev" | jq -r '.identifiers[0] // empty')
    [[ -z "$id" || "$(echo "$id" | jq -r '.[0]')" != "zha" ]] && continue
    
    mac=$(echo "$id" | jq -r '.[1]' | tr -d ':')
    name=$(echo "$dev" | jq -r '.name_by_user // .name' | sed "s/'/''/g")
    
    echo "'0x$mac':" >> devices.yaml
    echo "    friendly_name: '$name'" >> /homeassistant/zigbee2mqtt/devices.yaml
done
```

## Post Migrating

Start Zigbee2MQTT, your devices should be migrated, but you need to interview them again:

> Since zigbee2mqtt has no information about devices yet, they all are in unsupported state.
> I then requested interview for each device. All details are retrieved from the device and it starts operating in Z2M.
> 
> The usual wisdom applies: first interview mains-powered devices that you know are routers, then all mains powered devices, then battery-powered ones. You will have to go and wake up each battery-powered device.

If everything worked, delete ZHA. If you want, you can also delete `yq` with `rm /usr/bin/yq`.

## Sources

- https://github.com/Koenkk/zigbee2mqtt/discussions/24478#discussioncomment-11063040
- https://github.com/Koenkk/zigbee2mqtt/discussions/24478#discussioncomment-11728487
- https://github.com/Koenkk/zigbee2mqtt/discussions/24478#discussioncomment-11798112
