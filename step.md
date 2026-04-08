## Yangi switch qo'shish — to'liq ketma-ketlik

---

### 1. Switch'ni SNMP uchun tayyorlash

```bash
# Switch'ga SSH bilan kiring va SNMP sozlang
# Eltex misoli:
snmp-server community public ro
snmp-server host 10.10.10.1 traps version 2c public
snmp-server enable traps

# Cisco misoli:
snmp-server community public RO
snmp-server host 10.10.10.1 version 2c public
snmp-server enable traps snmp linkdown linkup coldstart
```

---

### 2. Switch'dan ma'lumot kelayotganini tekshirish

```bash
# sysDescr — switch ishlayaptimi?
snmpget -v 2c -c public 192.168.1.1 1.3.6.1.2.1.1.1.0

# sysObjectID — vendor OID prefiksi
snmpget -v 2c -c public 192.168.1.1 1.3.6.1.2.1.1.2.0

# Interface ro'yxati
snmpwalk -v 2c -c public 192.168.1.1 1.3.6.1.2.1.2.2.1.2

# Interface status
snmpwalk -v 2c -c public 192.168.1.1 1.3.6.1.2.1.2.2.1.8

# HC counters (64-bit) bor yoki yo'q
snmpwalk -v 2c -c public 192.168.1.1 1.3.6.1.2.1.31.1.1.1.6
```

---

### 3. Vendor aniqlaymiz

```bash
# sysObjectID natijasidan vendor OID prefiksi olinadi
# Masalan: iso.3.6.1.4.1.XXXXX → XXXXX raqami

# Mashhur vendor prefikslari:
# 9     → Cisco
# 2636  → Juniper
# 14988 → MikroTik
# 2011  → Huawei
# 35265 → Eltex
# 11    → HP/Aruba
# 25506 → H3C
# 6486  → Alcatel
```

---

### 4. Vendor-specific OID topamiz

```bash
# CPU OID — Zabbix template'dan yoki vendor hujjatidan
snmpwalk -v 2c -c public 192.168.1.1 1.3.6.1.4.1.XXXXX | grep -i cpu

# Memory OID
snmpwalk -v 2c -c public 192.168.1.1 1.3.6.1.4.1.XXXXX | grep -i mem

# Yoki Zabbix rasmiy template'laridan olish:
# https://git.zabbix.com/projects/ZBX/repos/zabbix/browse/templates/net
```

---

### 5. Pipeline fayllarini yangilash

**`01_snmp_discovery.conf`** — vendor regex qo'shish:

```ruby
# Hosts ga yangi switch qo'shing
hosts => [
  { host => "udp:10.10.11.53/161" community => "sw2"    version => "2c" retries => 2 timeout => 5000 }
  { host => "udp:192.168.1.1/161" community => "public" version => "2c" retries => 2 timeout => 5000 }
]

# Vendor detection ga yangi vendor qo'shing
} else if [1.3.6.1.2.1.1.2.0] =~ /1\.3\.6\.1\.4\.1\.XXXXX\./ {
  mutate { replace => { "vendor" => "yangi_vendor" } }
}
```

**`02_snmp_critical.conf`** — CPU/Memory qo'shish:

```ruby
# Hosts ga qo'shing
{ host => "udp:192.168.1.1/161" community => "public" version => "2c" retries => 2 timeout => 5000 }

# GET ga CPU/Memory OID qo'shing
get => [
  ...
  "1.3.6.1.4.1.XXXXX.cpu.oid",    # yangi vendor CPU
  "1.3.6.1.4.1.XXXXX.memory.oid"  # yangi vendor Memory
]

# Filter ga vendor blok qo'shing
} else if [vendor] == "yangi_vendor" {
  mutate { rename => { "1.3.6.1.4.1.XXXXX.cpu.oid" => "device.cpu.util_pct" } }
}
```

**`03_snmp_normal.conf`** — faqat hosts ga qo'shish yetarli:

```ruby
# Standart IF-MIB barcha vendorda bir xil
hosts => [
  ...
  { host => "udp:192.168.1.1/161" community => "public" version => "2c" retries => 2 timeout => 5000 }
]
```

**`04_snmp_inventory.conf`** — vendor serial/model OID qo'shish:

```ruby
get => [
  ...
  "1.3.6.1.4.1.XXXXX.serial.oid"  # yangi vendor serial
]

} else if [vendor] == "yangi_vendor" {
  mutate { rename => { "1.3.6.1.4.1.XXXXX.serial.oid" => "device.serial_number" } }
}
```

**`05_snmp_trap.conf`** — vendor trap OID qo'shish:

```ruby
when /1\.3\.6\.1\.4\.1\.XXXXX\./
  event.set("event.action",   "yangi_vendor-trap")
  event.set("vendor",         "yangi_vendor")
  event.set("event.severity", "medium")
```

---

### 6. Trap yo'naltirishni sozlash

```bash
# Switch Logstash serveriga trap yuborishini tekshirish
# Server tomonda:
tcpdump -i any udp port 162 -n

# Switch portini o'chir-yoq qilib trap kelayotganini ko'rish
```

---

### 7. Logstash hot reload tekshirish

```bash
# Fayl saqlanishi bilanoq Logstash o'zi qayta yuklaydi
# Tekshirish:
docker logs logstash 2>&1 | grep "reload\|started" | tail -5
```

---

### 8. Ma'lumot kelayotganini tekshirish

```bash
# Discovery (vendor to'g'ri aniqlanganmi?)
docker exec logstash cat /usr/share/logstash/snmp-cache/vendor_cache.json

# Critical (interface status kelayaptimi?)
curl -s -u elastic:${ELASTIC_PASSWORD} \
  "http://localhost:9200/metrics-snmp.critical-production/_search" \
  -H "Content-Type: application/json" \
  -d '{
    "query": {"term": {"device.ip": "192.168.1.1"}},
    "size": 1,
    "sort": [{"@timestamp": {"order": "desc"}}]
  }' | python3 -m json.tool

# Trap (trap kelayaptimi?)
curl -s -u elastic:${ELASTIC_PASSWORD} \
  "http://localhost:9200/logs-snmp.trap-production/_search" \
  -H "Content-Type: application/json" \
  -d '{
    "query": {"term": {"device.ip": "192.168.1.1"}},
    "size": 3,
    "sort": [{"@timestamp": {"order": "desc"}}]
  }' | python3 -m json.tool
```

---

### Qisqa chizma

```
Yangi switch
    │
    ▼
1. SNMP sozlash (switch tomonda)
    │
    ▼
2. snmpget/snmpwalk bilan tekshirish
    │
    ▼
3. sysObjectID → vendor OID prefiksi aniqlash
    │
    ▼
4. Vendor CPU/Memory OID topish (Zabbix template yoki hujjat)
    │
    ▼
5. 5 ta pipeline faylga hosts + vendor OID qo'shish
    │
    ▼
6. Hot reload — restart shart emas
    │
    ▼
7. vendor_cache.json tekshirish
    │
    ▼
8. ES da ma'lumot kelganini tasdiqlash
```
