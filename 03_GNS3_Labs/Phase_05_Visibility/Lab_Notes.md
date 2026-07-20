---
title: "Phase 05: Visibility & Hardening"
tags:
  - lab/visibility
  - lab/hardening
status: "draft"
---

# Phase 05: Visibility, Syslog & Hardening

> **Note:** Updated for current topology. FortiAnalyzer is excluded from design — logging uses Alpine + socat as syslog receiver (UDP 514). Grafana/Prometheus provides visualization.

## Syslog Forwarding

Traffic Gen + Syslog container listens on UDP 514 via socat. Configure on both FGTs:

```fortinet
config log syslogd setting
    set status enable
    set server 192.168.20.50
    set port 514
    set format default
end
```

## Administrative Hardening

```fortinet
config system admin
    edit "admin"
        set trusthost1 192.168.10.0 255.255.255.0
        set trusthost2 192.168.20.0 255.255.255.0
    next
end
```

## Port Hardening

```fortinet
config system global
    set admin-sport 10443
    set admin-ssh-port 10022
    set admin-https-ssl-versions tls1-2 tls1-3
    set admin-lockout-threshold 5
    set admin-lockout-duration 10
end
```

## Grafana Dashboards

Monitoring stack (Grafana + Prometheus) on `192.168.20.60:3000`. Configure:

1. Prometheus scrapes FGT metrics via SNMP exporter
2. Grafana datasource → Prometheus
3. Pre-built dashboard: live sessions, throughput, blocked attacks, VPN status
