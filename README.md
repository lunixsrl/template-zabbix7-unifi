# Zabbix Template for UniFi Network Application

Zabbix 7 template for monitoring UniFi Access Points through the local UniFi Network Application API.

The template is intended for self-hosted UniFi Network Application installations and supports multiple UniFi sites from a single controller. Sites and access points are discovered automatically.

Tested with:

- Zabbix 7.0.x
- UniFi Network Application 10.x
- Self-hosted UniFi Network Application
- Zabbix Server and Zabbix Proxy

## Features

- Multi-site discovery
- Automatic AP discovery
- One API login and collection cycle for all visible sites
- Site-aware item and trigger names
- Site/AP tags for filtering in Zabbix
- No SNMP or Zabbix Agent required on the APs

The template currently collects:

- AP status
- IP address
- Model
- Firmware version
- Uptime
- Connected clients
- WiFi satisfaction
- 2.4 GHz channel
- 2.4 GHz channel utilization
- 2.4 GHz TX power
- 2.4 GHz TX retries
- 2.4 GHz clients
- 5 GHz channel
- 5 GHz channel utilization
- 5 GHz TX power
- 5 GHz TX retries
- 5 GHz clients

Included trigger prototypes:

- AP offline for 5 minutes
- High 2.4 GHz channel utilization
- High 5 GHz channel utilization
- High 2.4 GHz TX retries
- High 5 GHz TX retries
- Low WiFi satisfaction
- Recently restarted AP

## Requirements

- Zabbix 7.0 or newer
- Self-hosted UniFi Network Application
- A local UniFi user with read-only access to the sites that should be monitored
- TCP/8443 connectivity from the Zabbix Server or Zabbix Proxy performing the checks to the UniFi Network Application

No host interface is required in Zabbix. The template uses a Zabbix Script item and the UniFi API directly.

## Installation

1. Download `template_unifi_network_application.yaml`.
2. In Zabbix, go to **Data collection → Templates → Import**.
3. Import the template.
4. Create a host for the UniFi Network Application.
5. Link the template **UniFi Network Application by API**.
6. If the controller should be queried by a Zabbix Proxy, assign the host to that proxy.
7. Configure the required host macros.
8. Do not add an Agent or SNMP interface unless you need it for something unrelated to this template.

## UniFi user

Create a dedicated local user in UniFi Network Application for monitoring.

Read-only access is sufficient. Grant the user access to every UniFi site that should appear in Zabbix.

The template calls:

```text
/api/login
/api/self/sites
/api/s/<site>/stat/device
```

`/api/self/sites` is used to discover all sites visible to the monitoring account. The template then queries each site and discovers devices with type `uap`.

Because site discovery is automatic, a separate template or Zabbix host is not required for every UniFi site.

## Zabbix macros

Configure these macros on the Zabbix host:

| Macro | Example | Description |
| --- | --- | --- |
| `{$UNIFI.URL}` | `https://192.168.1.10:8443` | Base URL of UniFi Network Application, without a trailing slash |
| `{$UNIFI.USER}` | `zabbix-monitor` | Local UniFi monitoring user |
| `{$UNIFI.PASSWORD}` | `********` | Password for the monitoring user |

`{$UNIFI.PASSWORD}` is defined as a Zabbix Secret text macro.

There is no site macro. All sites visible to the UniFi monitoring user are discovered automatically.

## Multi-site discovery

For every discovered AP, the template adds the UniFi site ID and site name to the discovery data.

Items are named using the site description, for example:

```text
[Head Office] AP AP-01: Status
[Branch Office] AP AP-02: Clients
```

Triggers also contain the site name:

```text
UniFi [Branch Office] AP AP-02: Offline for 5 minutes
```

Discovered items and triggers are tagged with the site and AP name. Trigger prototypes also include the AP MAC address. This makes it possible to filter Zabbix Problems and dashboards by site.

## How it works

The master Script item runs once per minute.

On every execution it:

1. Authenticates against the local UniFi API.
2. Retrieves the list of sites available to the monitoring user.
3. Queries `/stat/device` for every site.
4. Keeps UniFi AP (`uap`) devices.
5. Adds the site ID and site description to every AP object.
6. Returns a single JSON document to Zabbix.

A low-level discovery rule processes this JSON and creates dependent items for every AP.

This avoids making a separate API request for every metric.

## Notes

### Zabbix Proxy

When a host is assigned to a Zabbix Proxy, the API request is executed by that proxy.

Make sure the proxy can reach the value configured in `{$UNIFI.URL}`.

For example:

```bash
curl -k https://192.168.1.10:8443/
```

### Site permissions

If a site does not appear in Zabbix, check the permissions of the UniFi monitoring user first.

Only sites returned by `/api/self/sites` can be discovered.

### WiFi satisfaction

Some APs may report `-1` when satisfaction data is not available. The included satisfaction trigger ignores negative values.

### Radio data

The current template handles the UniFi `ng` and `na` radio entries as 2.4 GHz and 5 GHz respectively.

6 GHz radio metrics are not currently included.

## Current scope

This template currently focuses on UniFi Access Points.

It does not currently discover or monitor:

- UniFi switches
- UniFi gateways
- Client devices
- WLAN/SSID statistics
- 6 GHz radio statistics

These may be added later.

## Compatibility

The template was developed and tested against UniFi Network Application 10.3.58 and Zabbix 7.0.30.

It uses the local UniFi Network Application API. These endpoints are not guaranteed to remain unchanged between UniFi releases, so test the template after major UniFi upgrades.

UniFi OS consoles and Cloud Gateways have not been tested with this template.

## Security

Use a dedicated read-only UniFi account for monitoring.

Do not store real credentials in the template file or commit host-specific Zabbix configuration containing credentials to a public repository.

The template itself contains no controller IP addresses, site IDs, customer names or credentials.
