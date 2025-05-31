# Migrate Home Assistant ZHA to Zigbee2MQTT

## !!! DISCLAIMER !!!

**This worked for me and hopefully will work for you, but MAKE BACKUPS!\
\
If you don't understand any of the code below, DON'T run this!\
\
Also don't do this when you are in a hurry — this might completely destroy your Home Assistant ZHA setup.\
You have been warned... watch out for dragons...**

------------------

### Thanks to
- [seidtgeist](https://github.com/seidtgeist)
- [toine512](https://github.com/toine512)
- [teal-bauer](https://github.com/teal-bauer)

whose work this is based on.

## Guide

This script will migrate your [ZHA](https://www.home-assistant.io/integrations/zha/) devices to [Zigbee2MQTT](https://www.zigbee2mqtt.io/) using the content of:
- `/homeassistant/.store/core.device_registry`
- `/homeassistant/zigbee.db`
  
It will generate/edit:
- `/homeassistant/zigbee2mqtt/devices.yaml`
- `/homeassistant/zigbee2mqtt/configuration.yaml`
- `/homeassistant/zigbee2mqtt/database.db`

This is tested on Home Assistant Operating System:
- Z2M: 2.1.1-1
- HA-Core: 2024.12.5
- HA-OS: 14.1

On other Home Assistant setups this will only work **with** modifications!

### Before Migration

For this to work, you need to:
 - [Install SSH & Web Terminal](https://community.home-assistant.io/t/home-assistant-community-add-on-ssh-web-terminal/33820)
 - [Install Zigbee2MQTT](https://github.com/zigbee2mqtt/hassio-zigbee2mqtt#installation), **BUT** don't start it!
 - [Install Mosquitto](https://github.com/home-assistant/addons/blob/master/mosquitto/DOCS.md)
 - [Deactivate ZHA](https://community.home-assistant.io/t/how-to-disable-zha-zigbee-home-automation/553061/21)
 - Start Zigbee2MQTT
 - Check the Protocol and make sure it is fully started
 - Stop Zigbee2MQTT

Afterwards, connect to home assistant via `ssh` or open the web terminal and run the commands below:

### Migration

```shell
set -euo pipefail

# On my instalation the tables got postfix version strings like: _v11 / _v12 / _v13.
# We need to get the latest version, if no postfix is found return '':
VERSION=$(sqlite3 /homeassistant/zigbee.db "SELECT COALESCE('_v' || MAX(CAST(substr(name, instr(name, '_v') + 2) AS INT)), '') FROM sqlite_master WHERE name LIKE '%_v%' AND type='table';")

# Append the Z2M Database:
sqlite3 /homeassistant/zigbee.db "SELECT CONCAT('{"'"'"id"'"'": ', (ROW_NUMBER() OVER (ORDER BY d.ieee)) + 1, ', "'"'"type"'"'": "'"'"EndDevice"'"'", "'"'"ieeeAddr"'"'": "'"'"0x', REPLACE(d.ieee, ':', ''), '"'"'", "'"'"nwkAddr"'"'": ', d.nwk, '}') FROM devices${VERSION} d JOIN endpoints${VERSION} e ON d.ieee = e.ieee ;" >> /homeassistant/zigbee2mqtt/database.db

# Get the ZHA Zigbee config:
ZIGBEE_CONF=$(sqlite3 /homeassistant/zigbee.db "SELECT backup_json FROM network_backups${VERSION} ORDER BY id DESC LIMIT 1 ;" | jq -c)

# Parse and transform ZHA config:
pan_id=$(echo "$ZIGBEE_CONF" | jq -r '.network_info.pan_id' | awk '{print "0x" $1}')
ext_pan_id=$(echo "$ZIGBEE_CONF" | jq -r '.network_info.extended_pan_id' | awk -F: '{for (i=8; i>0; i--) printf "0x%s%s", $i, (i>1?", ":"")}')
channel=$(echo "$ZIGBEE_CONF" | jq -r '.network_info.channel')
network_key=$(echo "$ZIGBEE_CONF" | jq -r '.network_info.network_key.key' | awk -F: '{for (i=1; i<=NF; i++) printf "0x%s%s", $i, (i<NF?", ":"")}')

# Delete the advanced section from the default Z2M configuration
sed -i '/^\s*advanced:/,/^[^[:space:]]/ { /^[^[:space:]]/!d; /^\s*advanced:/d }' /homeassistant/zigbee2mqtt/configuration.yaml

# Append the Z2M config:
cat <<EOF >> /homeassistant/zigbee2mqtt/configuration.yaml
advanced:
  pan_id: $pan_id
  ext_pan_id: [$ext_pan_id]
  channel: $channel
  network_key: [$network_key]
devices: devices.yaml
EOF

# Create the friendly device names for Z2M:
jq -c '.data.devices[]' /homeassistant/.storage/core.device_registry | while read -r dev; do
    id=$(echo "$dev" | jq -r '.identifiers[0] // empty')
    [[ -z "$id" || "$(echo "$id" | jq -r '.[0]')" != "zha" ]] && continue
    
    mac=$(echo "$id" | jq -r '.[1]' | tr -d ':')
    name=$(echo "$dev" | jq -r '.name_by_user // .name' | sed "s/'/''/g")
    
    echo "'0x$mac':" >> /homeassistant/zigbee2mqtt/devices.yaml
    echo "    friendly_name: '$name'" >> /homeassistant/zigbee2mqtt/devices.yaml
done
```

### If anything went wrong:

- **DON'T** start Zigbee2MQTT
- Read the output!
- Read this guide again!

If you can't find the failure:
- uninstall / deactivate Zigbee2MQTT
- restart Home Assistant

ZHA should still be working.

### After Migration

Start Zigbee2MQTT, your devices should be migrated, but you need to interview them again:

> Since zigbee2mqtt has no information about devices yet, they all are in unsupported state.
> I then requested interview for each device. All details are retrieved from the device and it starts operating in Z2M.
> 
> The usual wisdom applies: first interview mains-powered devices that you know are routers, then all mains powered devices, then battery-powered ones. You will have to go and wake up each battery-powered device.

If everything worked, delete ZHA.

## Sources

- https://github.com/Koenkk/zigbee2mqtt/discussions/24478#discussioncomment-11063040
- https://github.com/Koenkk/zigbee2mqtt/discussions/24478#discussioncomment-11728487
- https://github.com/Koenkk/zigbee2mqtt/discussions/24478#discussioncomment-11798112
