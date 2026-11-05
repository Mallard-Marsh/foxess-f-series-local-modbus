# Fox ESS F-series (F6000) — local Modbus TCP over the stock Wi-Fi logger

Reading a **Fox ESS F6000** (solar-only, single-phase) **locally** in Home Assistant, using the Wi-Fi data logger that came with the inverter. No RS485 adapter, no cloud, and no custom integration: just Home Assistant's built-in [`modbus`](https://www.home-assistant.io/integrations/modbus/) integration.

I couldn't find anyone who had done this with an F-series before. The [F5000 discussion on foxess_modbus](https://github.com/nathanmarlor/foxess_modbus/discussions/220) concluded that the E/F series had no Modbus interface. On my setup it does, through the logger.

## Tested on

| | |
|---|---|
| Inverter | 2 × Fox ESS **F6000**, firmware Master 1.03 / Slave 1.00 / ARM 1.01 |
| Logger | Stock Fox Wi-Fi data logger (USB-A stick), firmware **2.05**, Espressif-based (MAC prefix `88:13:BF`) |
| Home Assistant | 2026.10, core `modbus` integration |
| Date | November 2026, UK |

## What works

- **Modbus TCP, port 502** on the logger's Wi-Fi IP address
- **Unit / slave ID 1.** Not 247, which the Smart/WL hybrid loggers use.
- **Input registers (function 4)** in Fox's newer **39xxx** map, the same addresses the H1 G2 / H3 Pro / KH use. Function 3 returns the same values.
- Unmapped addresses **don't return an exception; the logger just doesn't answer.** A scan that hits a gap looks like a timeout. Read in small blocks and open a fresh connection after a timeout.

## Confirmed registers

Every value was checked against FoxCloud at the same moment.

| Register | Type | Scale | Meaning | Example |
|---|---|---|---|---|
| 39000 | uint16 | — | Protocol version? (`0x0101`) | 257 |
| 39018–39025 | ASCII, 2 chars/reg | — | Inverter serial number | (matches the label) |
| 39054 | uint16 | 1 W | Rated power | 6000 |
| 39070 | uint16 | 0.1 V | PV1 voltage | 2036 → 203.6 V |
| 39071 | uint16 | 0.01 A | PV1 current | 180 → 1.80 A |
| 39072 | uint16 | 0.1 V | PV2 voltage | 1416 → 141.6 V |
| 39073 | uint16 | 0.01 A | PV2 current | 30 → 0.30 A |
| 39118–39119 | int32 (hi, lo) | 1 W | PV input power | 406 W |
| 39123 | uint16 | 0.1 V | Grid voltage | 2457 → 245.7 V |
| 39127 | uint16 | 0.001 A | AC output current | 1610 → 1.61 A |
| 39134–39135 | int32 (hi, lo) | 1 W | **AC active power** | 395 W |
| 39136–39137 | int32 (hi, lo) | 1 var | Reactive power | −39 var |
| 39138 | uint16 | 0.001 | Power factor | 994 → 0.994 |
| 39139 | uint16 | 0.01 Hz | Grid frequency | 5000 → 50.00 Hz |
| 39141 | uint16 | 0.1 °C | Inverter temperature | 120 → 12.0 °C |
| 39150–39152 | ? | ? | Status/flags? Unknown | 32868, 0, 20 |

Readable blocks found: **39000–39164**. I couldn't find **energy totals** (daily/lifetime kWh) anywhere I scanned (37000–40000 input; 40000–42000 and 46000–46200 holding). I count kWh in HA with an `integration` (Riemann sum) sensor instead.

I only read registers. I didn't try any writes and I'm not suggesting anyone does.

## Gotchas

1. **The logger accepts one Modbus TCP connection at a time.** While Home Assistant is connected, the port looks closed to everything else. Don't point two tools at it.
2. **Router "device isolation" will hide it completely.** One of my two loggers had the router's Device Isolation (client isolation) switched on. It still reached FoxCloud, but no ping and no ports from the LAN. Turning isolation off fixed it.
3. **The logger sleeps at night.** F-series inverters power the logger, so it drops off Wi-Fi after dark. Expect `unavailable` overnight.
4. **Reserve its IP** in your router. Mine moved address between days.
5. Occasional `transaction_id` mismatch messages from pymodbus are harmless, because the next poll recovers. A 15 s scan interval and `delay: 1` keep them rare.
6. FoxCloud keeps working alongside local polling.

## Home Assistant config

See [`fox_f6000_modbus.yaml`](fox_f6000_modbus.yaml). Copy it into `configuration.yaml`, change the IP, and restart. For a second inverter, duplicate the hub with its own logger IP and new names/unique IDs.

## Quick test without Home Assistant

```bash
python3 - <<'EOF'
import socket, struct
s = socket.create_connection(("LOGGER_IP", 502), 4); s.settimeout(3)
s.send(struct.pack(">HHHBBHH", 1, 0, 6, 1, 4, 39070, 4))  # unit 1, FC4, PV1/PV2 V+A
r = s.recv(256); v = struct.unpack(">4H", r[9:17])
print(f"PV1 {v[0]/10} V {v[1]/100} A | PV2 {v[2]/10} V {v[3]/100} A")
EOF
```

If that connects but times out, check that nothing else is already connected (gotcha 1) and that it's daytime (gotcha 3).

## Unknowns / help wanted

- Do other F-series models (F3000–F5000) and E-series behave the same?
- Does it depend on logger firmware 2.05?
- Is there an energy-total register somewhere outside the ranges I scanned?
- What do 39150–39152 mean?

Issues and PRs are welcome.
