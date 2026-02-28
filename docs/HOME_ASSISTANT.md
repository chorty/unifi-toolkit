# Home Assistant Integration

Pull data from UniFi Toolkit into Home Assistant using REST sensors. No custom integration required.

## Setup

Add the following to your Home Assistant `configuration.yaml`. Replace `TOOLKIT_IP:8000` with your toolkit's IP and port.

---

## 1. Network Pulse

### Gateway Health (CPU, RAM, uptime, WAN status)

```yaml
rest:
  - resource: http://TOOLKIT_IP:8000/pulse/api/stats/gateway
    scan_interval: 60
    sensor:
      - name: "UniFi Gateway CPU"
        value_template: "{{ value_json.cpu_utilization | default(0) | round(1) }}"
        unit_of_measurement: "%"
        icon: mdi:cpu-64-bit

      - name: "UniFi Gateway RAM"
        value_template: "{{ value_json.mem_utilization | default(0) | round(1) }}"
        unit_of_measurement: "%"
        icon: mdi:memory

      - name: "UniFi Gateway Uptime"
        value_template: >-
          {% set u = value_json.uptime | default(0) | int %}
          {{ '%dd %dh %dm' | format(u // 86400, (u % 86400) // 3600, (u % 3600) // 60) }}
        icon: mdi:clock-check

      - name: "UniFi WAN Status"
        value_template: "{{ value_json.wan_status | default('unknown') }}"
        icon: mdi:wan

      - name: "UniFi WAN IP"
        value_template: "{{ value_json.wan_ip | default('unknown') }}"
        icon: mdi:ip-network
```

### Device Counts

```yaml
  - resource: http://TOOLKIT_IP:8000/pulse/api/stats/devices
    scan_interval: 120
    sensor:
      - name: "UniFi Total Clients"
        value_template: "{{ value_json.clients | default(0) }}"
        icon: mdi:devices

      - name: "UniFi Wireless Clients"
        value_template: "{{ value_json.wireless_clients | default(0) }}"
        icon: mdi:wifi

      - name: "UniFi Wired Clients"
        value_template: "{{ value_json.wired_clients | default(0) }}"
        icon: mdi:ethernet

      - name: "UniFi Access Points"
        value_template: "{{ value_json.aps | default(0) }}"
        icon: mdi:access-point

      - name: "UniFi Switches"
        value_template: "{{ value_json.switches | default(0) }}"
        icon: mdi:switch
```

### Current Throughput

```yaml
  - resource: http://TOOLKIT_IP:8000/pulse/api/stats/bandwidth
    scan_interval: 30
    sensor:
      - name: "UniFi Download Rate"
        value_template: "{{ value_json.current_rx_rate | default(0) }}"
        unit_of_measurement: "Mbps"
        icon: mdi:download

      - name: "UniFi Upload Rate"
        value_template: "{{ value_json.current_tx_rate | default(0) }}"
        unit_of_measurement: "Mbps"
        icon: mdi:upload
```

---

## 2. Threat Watch

### Threat Statistics

```yaml
  - resource: http://TOOLKIT_IP:8000/threats/api/events/stats?time_range=24h
    scan_interval: 300
    sensor:
      - name: "UniFi Threats 24h"
        value_template: "{{ value_json.events_24h | default(0) }}"
        icon: mdi:shield-alert

      - name: "UniFi Threats 7d"
        value_template: "{{ value_json.events_7d | default(0) }}"
        icon: mdi:shield-alert-outline

      - name: "UniFi Threats Blocked"
        value_template: "{{ value_json.blocked_count | default(0) }}"
        icon: mdi:shield-check

      - name: "UniFi Threats Total"
        value_template: "{{ value_json.total_events | default(0) }}"
        icon: mdi:shield-bug

      - name: "UniFi Top Attacker"
        value_template: >
          {% if value_json.top_attackers | default([]) | length > 0 %}
            {{ value_json.top_attackers[0].ip }}
          {% else %}
            none
          {% endif %}
        icon: mdi:account-alert

      - name: "UniFi Top Attacker Count"
        value_template: >
          {% if value_json.top_attackers | default([]) | length > 0 %}
            {{ value_json.top_attackers[0].count }}
          {% else %}
            0
          {% endif %}
        icon: mdi:counter
```

### Recent Events (latest 5)

```yaml
  - resource: http://TOOLKIT_IP:8000/threats/api/events?page_size=5&sort=timestamp&sort_direction=desc
    scan_interval: 300
    sensor:
      - name: "UniFi Latest Threat"
        value_template: >
          {% if value_json.events | default([]) | length > 0 %}
            {{ value_json.events[0].signature | default('unknown') | truncate(50) }}
          {% else %}
            none
          {% endif %}
        icon: mdi:alert-circle

      - name: "UniFi Latest Threat Source"
        value_template: >
          {% if value_json.events | default([]) | length > 0 %}
            {{ value_json.events[0].src_ip | default('unknown') }}
          {% else %}
            none
          {% endif %}
        icon: mdi:ip-network

      - name: "UniFi Latest Threat Severity"
        value_template: >
          {% if value_json.events | default([]) | length > 0 %}
            {{ value_json.events[0].severity | default(0) }}
          {% else %}
            0
          {% endif %}
        icon: mdi:alert
```

---

## 3. WiFi Stalker (Presence Detection)

### All Tracked Devices

```yaml
  - resource: http://TOOLKIT_IP:8000/stalker/api/devices
    scan_interval: 60
    sensor:
      - name: "UniFi Tracked Devices Online"
        value_template: >
          {{ value_json.devices | default([]) | selectattr('is_connected', 'equalto', true) | list | length }}
        icon: mdi:account-check

      - name: "UniFi Tracked Devices Total"
        value_template: "{{ value_json.total | default(0) }}"
        icon: mdi:account-group
```

### Individual Device Presence (per tracked device)

To track a specific person/device, first find the device ID from the WiFi Stalker UI, then add:

```yaml
  - resource: http://TOOLKIT_IP:8000/stalker/api/devices/DEVICE_ID
    scan_interval: 60
    sensor:
      - name: "Matt Phone"
        value_template: >
          {% if value_json.is_connected | default(false) %}
            home
          {% else %}
            away
          {% endif %}
        icon: mdi:cellphone

      - name: "Matt Phone AP"
        value_template: "{{ value_json.current_ap_name | default('unknown') }}"
        icon: mdi:access-point

      - name: "Matt Phone Signal"
        value_template: "{{ value_json.current_signal_strength | default(0) }}"
        unit_of_measurement: "dBm"
        icon: mdi:signal

      - name: "Matt Phone Last Seen"
        value_template: "{{ value_json.last_seen | default('unknown') }}"
        icon: mdi:clock-outline
```

> **Tip:** You can use this as a `device_tracker`-like entity for automations
> (e.g., arrive home → turn on lights).

---

## 4. Real-Time Alerts via Webhooks (Optional)

For **instant** threat notifications instead of polling, configure a webhook in Threat Watch or WiFi Stalker that sends to Home Assistant's webhook automation trigger.

### In Home Assistant — create an automation:

```yaml
automation:
  - alias: "UniFi Threat Alert"
    trigger:
      - platform: webhook
        webhook_id: unifi-threat-alert
        allowed_methods:
          - POST
    action:
      - service: notify.mobile_app_your_phone
        data:
          title: "🚨 Network Threat Detected"
          message: "{{ trigger.json.message | default('IPS event detected') }}"
          data:
            priority: high
```

### In UniFi Toolkit — add a webhook:

1. Go to **Threat Watch** → **Webhooks**
2. Add webhook URL: `http://YOUR_HA_IP:8123/api/webhook/unifi-threat-alert`
3. Type: **Generic**
4. Enable desired event types

Same pattern works for WiFi Stalker connect/disconnect events.

---

## 5. Dashboard Card Examples

### Lovelace Entities Card

```yaml
type: entities
title: UniFi Network
entities:
  - entity: sensor.unifi_gateway_cpu
  - entity: sensor.unifi_gateway_ram
  - entity: sensor.unifi_wan_status
  - entity: sensor.unifi_total_clients
  - entity: sensor.unifi_access_points
  - entity: sensor.unifi_switches
  - entity: sensor.unifi_download_rate
  - entity: sensor.unifi_upload_rate
  - entity: sensor.unifi_threats_24h
  - entity: sensor.unifi_threats_blocked
  - entity: sensor.unifi_tracked_devices_online
```

### Lovelace Gauge Card (Gateway CPU)

```yaml
type: gauge
entity: sensor.unifi_gateway_cpu
name: Gateway CPU
min: 0
max: 100
severity:
  green: 0
  yellow: 60
  red: 85
```

---

## Notes

- **Polling intervals:** Adjust `scan_interval` (seconds) based on your needs. Gateway/throughput can poll faster (30-60s); threats can poll slower (300s).
- **No authentication required** when running in `local` deployment mode. For `production` mode, you'll need to add authentication headers.
- **Network:** Home Assistant must be able to reach the toolkit's IP and port.
