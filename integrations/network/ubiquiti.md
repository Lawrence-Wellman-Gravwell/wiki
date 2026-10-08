# Ubiquiti UniFi

:::{csv-table}
:align: left
:width: 45%
:widths: 15, 25
**Integration Details**
    Ingester, [Simple Relay](/ingesters/simple_relay)
    Preprocessor, [Regex Router](/ingesters/preprocessors/regexrouter.md)
:::

## Ubiquiti UniFi Configuration

The UniFi Network application can export system, security, and client-activity logs. See Ubiquiti's documentation: [UniFi System Logs & SIEM Integration](https://help.ui.com/hc/en-us/articles/33349041044119).

1. Open the UniFi Network application and go to **Integration > System Logging / SIEM** (some versions place this under **Settings > CyberSecure > Traffic Logging > Activity Logging (Syslog)**).
2. Select **SIEM Server** as the destination.
3. Choose the log categories to export: Monitoring, Internet, Power, Security (Firewall, Honeypot, Intrusion Prevention), and System (Admin Activity, Devices, Network, VPN).
4. Enter the IP address of your Gravwell Simple Relay ingester and the port of its listener (`15130` in the sample below).

## Gravwell Configuration

### Gravwell Storage Well Configuration

Set up the well configuration in your Gravwell indexers.

**Sample well config:**  
Create or edit: `/opt/gravwell/etc/gravwell.conf.d/ubiquiti-well.conf`
```ini
[Storage-Well "ubiquiti"]
    Location=/opt/gravwell/storage/ubiquiti
    Tags=ubiquiti*
```

### Gravwell Ingester Configuration: Simple Relay

UniFi sends logs in several formats depending on the log types you choose. The preprocessors below route each format to its own tag:

| Tag | Contents |
|---|---|
| `ubiquiti-cef` | UniFi Network application events in CEF |
| `ubiquiti-ap` | UniFi access points (UAP, U6 and U7 models) |
| `ubiquiti-switch` | UniFi switches (USW models) |
| `ubiquiti-udm` | UDM Pro operating system and daemon logs |
| `ubiquiti-firewall` | UDM firewall rule logs |
| `ubiquiti` | Anything that matches none of the above |

Entries that match no preprocessor, such as a new device type or a changed format, stay in the `ubiquiti` tag, so no data is lost.

**Sample Ubiquiti config:**  
Create or edit: `/opt/gravwell/etc/simple_relay.conf.d/ubiquiti.conf`
```ini
# UniFi Network application CEF export (BSD-style header with an optional PRI, then CEF:0|Vendor|Product|...).
[preprocessor "ubnt-cef"]
    Type = regexrouter
    Drop-Misses = false
    Regex = `^(?:<\d+>)?\w{3}\s+\d+\s+[\d:]{8}\s+\S+\s+CEF:\d+\|(?P<vendor>[^|]+)\|`
    Route-Extraction = vendor
    Route = Ubiquiti:ubiquiti-cef

# UniFi network devices: "<host> <12-hex MAC>,<MODEL>-<firmware>: message". APs include a
# BSD timestamp, some switches do not, so the timestamp is optional. The model family is the
# text before the first "-" or "_" (UAP-nanoHD -> UAP, USW_FLEX_MINI -> USW).
[preprocessor "ubnt-device"]
    Type = regexrouter
    Drop-Misses = false
    Regex = `^<\d+>(?:\w{3}\s+\d+\s+[\d:]{8}\s+)?\S+\s+[0-9a-fA-F]{12},(?P<family>[A-Za-z0-9]+)[-_]`
    Route-Extraction = family
    Route = UAP:ubiquiti-ap
    Route = U6:ubiquiti-ap
    Route = U7:ubiquiti-ap
    Route = USW:ubiquiti-switch

# UDM Pro OS logs: "<host> <host> program[pid]: message". The program name must not contain a
# comma or bracket, which keeps this from matching the firewall ("[RULE]") and device ("MAC,MODEL:") formats.
[preprocessor "ubnt-udm"]
    Type = regexrouter
    Drop-Misses = false
    Regex = `^<\d+>\w{3}\s+\d+\s+[\d:]{8}\s+(?P<host>\S+)\s+\S+\s+[^\s\[\],]+(?:\[\d+\])?:\s`
    Route-Extraction = host
    Route = UDMPro:ubiquiti-udm

# UDM firewall: "[RULESET-ACTION-n] [DESCR="..."] IN=<if> OUT=<if> ..." right after "host host".
# The capture is the literal "IN" so every rule set (LAN_LOCAL, WAN_IN, ...) routes the same way.
[preprocessor "ubnt-firewall"]
    Type = regexrouter
    Drop-Misses = false
    Regex = `^<\d+>\w{3}\s+\d+\s+[\d:]{8}\s+\S+\s+\S+\s+\[[^\]]+\]\s+(?:DESCR="[^"]*"\s+)?(?P<kind>IN)=\S*\s+OUT=`
    Route-Extraction = kind
    Route = IN:ubiquiti-firewall

[Listener "ubiquiti"]
    Bind-String="udp://0.0.0.0:15130"
    Tag-Name=ubiquiti
    Assume-Local-Timezone=true #if a time format does not have a timezone, assume local time
    Preprocessor=ubnt-cef
    Preprocessor=ubnt-device
    Preprocessor=ubnt-udm
    Preprocessor=ubnt-firewall
```

```{note}
Remember to restart the service to apply the new config:
`sudo systemctl restart gravwell_simple_relay.service`
```
