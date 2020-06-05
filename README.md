# Salt-formula-knot

This salt formula enables you to deploy [knot-dns](https://www.knot-dns.cz/) as your authoritative DNS Server.

## Example Pillars

### init.sls
```yaml
---
include:
{% if grains.fqdn.startswith('ns1.') %}
  - group.knot.master
{% else %}
  - group.knot.slave
{% endif %}

knot:
  server:
    params:
      rundir: /run/knot
      user: knot:knot
      listen: 
        - 0.0.0.0@53
        - ::@53

  log:
    syslog: 
      any: info

  # generate with `keymgr -t tsig_ffrn_ns_2020052100 hmac-sha384`
  key:
    tsig_key:
      algorithm: hmac-sha512
      secret: supersecretkey

  template:
    default:
      storage: "/var/lib/knot/zones"
      file: "%s.zone"
      journal-db: "/var/lib/knot/journal"
      kasp-db: "/var/lib/knot/keys"
      timer-db: "/var/lib/knot/timers"

  zone:
    example.org: {}
    test.example.org:
      file: "testone.zone"
```

### master.sls
```yaml
---
knot:
  zones-repository:
    remote: https://github.com/Freifunk-Rhein-Neckar/zones.git

  remote:
    remote_slave_ns:
      address:
        - "198.51.100.53"
        - "2001:db8:2::53"
        - "203.0.113.53"
        - "2001:db8:3::53"
      key: tsig_key

  acl:
    acl_slave_ns:
      address:
        - "198.51.100.53"
        - "2001:db8:2::53"
        - "203.0.113.53"
        - "2001:db8:3::53"
      key: tsig_key
      action: transfer

  template:
    default:
      notify: remote_slave_ns
      acl: acl_slave_ns
      zonefile-sync: -1
      zonefile-load: difference
      journal-content: changes
```

### slave.sls
```yaml
---
knot:
  remote:
    master_ns:
      address:
        - "192.0.2.53"
        - "2001:db8:1::53"
      key: tsig_key

  acl:
    acl_ns:
      address: 
        - "192.0.2.53"
        - "2001:db8:1::53"
      action: notify
      key: tsig_key

  template:
    default:
      master: master_ns
      acl: acl_ns
```
