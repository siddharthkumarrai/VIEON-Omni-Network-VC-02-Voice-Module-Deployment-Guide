# VIEON Omni-Network — VC-02 Voice Module Deployment Guide

**Target module:** AI-Thinker VC-02 (Unisound **US516P6** ASIC)
**Repo:** VIEON — Voice-Integrated Embodied Omni-Network
**Status:** Verified against official Ai-Thinker sources. Sections marked
`⚠️ UNVERIFIED` could not be confirmed and must be checked live on the
platform before you rely on them.

---

## ⚠️ Correction Notice

An earlier draft of this guide circulated with two serious errors that
would have broken the build if followed as-is:

1. **Wrong chip name.** It referred to "US551A." The real ASIC on the
   VC-02 is the **Unisound US516P6**. There is no public AI-Thinker /
   Unisound part called US551A — if your module's silkscreen says
   something different, photograph it and ask AI-Thinker support before
   flashing.
2. **Wrong UART command protocol.** It claimed the module sends plain
   ASCII strings like `WAKE_SUCCESS\n` over UART. The **factory
   protocol is a fixed 5-byte hex command code with an XOR checksum**,
   not ASCII text (see §3 below). This is confirmed directly from
   AI-Thinker's own published documentation. If your custom build turns
   out to also export ASCII (see the open question in §3.3), treat that
   as a platform-specific feature to verify, not the default behavior.

Everything below is rebuilt from verifiable sources: AI-Thinker's own
Medium tutorial series (Tutorials 1–3, by Ai-Thinker staff), the
Hackster.io article published by AI-Thinker's own account, and the
Mikroe-hosted VC-02 datasheet.

---

## 1. Platform & Account

- Platform: **http://voice.ai-thinker.com/**
- Create an account with your email.
- On the homepage, click **"Create Product"**.
- Fill in product name, select your scene/use case, and select module
  = **VC-02**.

From here the platform's settings page lets you configure, per
AI-Thinker's own tutorial:

- **Wake word(s)** — 3–6 syllable custom wake word, with a pronunciation
  scoring tool to check recognizability before you commit.
- **Wake-up reply** — up to 5 replies, played back randomly.
- **Command words + response words** — entered directly in a table on
  the platform.
- **Competing words** — words you deliberately mark as "not this," to
  reduce misrecognition between similar-sounding commands (e.g. so "36
  degrees" isn't misheard as "16 degrees").
- **Wake-up-free commands** — command words that trigger without a
  wake word first; combined wake words + wake-free commands cannot
  exceed 10 total.
- **Voice/speaker persona** — several built-in voices to choose from.

**⚠️ UNVERIFIED:** Exact field-by-field UI layout (exact button
positions, dropdown order) — Ai-Thinker's tutorial images weren't
retrievable in text form, only their captions/descriptions. Follow the
platform's own on-screen instructions; the structure above is confirmed,
the pixel-level layout is not.

---

## 2. Firmware Build & Extraction

Confirmed workflow (from AI-Thinker's own SDK documentation):

- After configuring your product, generate the SDK from the platform.
- Two firmware images come out of a build:

| File | Purpose | Burn method |
|---|---|---|
| `uni_app_release.bin` | Full image | **JTAG only** — dedicated AI-Thinker/Unisound JTAG debugger. **J-Link is explicitly not supported.** |
| `uni_app_release_update.bin` | Generated via the `build.sh update` command | **UART serial** — via `UniOneUpdateTool.exe` |

For a breadboard setup with no JTAG probe, you want the serial path:
**`uni_app_release_update.bin` + `UniOneUpdateTool.exe`**.

**⚠️ UNVERIFIED:** Exact build queue time, and the exact zip/rar file
naming convention — not confirmed from official sources, expect it to
vary by SDK version.

---

## 3. UART Protocol — This Is the Part to Get Right

### 3.1 Physical UART settings (confirmed, from AI-Thinker's own spec)

**UART1 (command/data interface):**
- Baud: **115200**, 8 data bits, 1 stop bit, no parity, no flow control (8N1)
- Pins: **TX1 / RX1** on the module

**UART0 (log output only, TX0 = IOB8):**
- Baud: **57600**, 8N1

### 3.2 Factory command code format (confirmed)

The module does **not** speak plain ASCII commands by default. Every
recognized command is emitted as a **fixed 5-byte frame**:

```
Byte 0: Start bit        — fixed 0x5A
Byte 1: Command number    — unique ID per command (e.g. 0x00 = wake word)
Byte 2: Reserved 1        — fixed 0x00
Byte 3: Reserved 2        — fixed 0x00
Byte 4: Check bit         — XOR of bytes 0-3
```

Example — wake word detected:
```
0x5A 0x00 0x00 0x00 0x5A
```
(check bit = 0x5A ^ 0x00 ^ 0x00 ^ 0x00 = 0x5A)

Checksum calculation in C, per AI-Thinker's own example:
```c
char start_bit     = 0x5A;
char cmd_num_bit    = 0x00;   // your assigned command number
char temp_num1_bit  = 0x00;
char temp_num2_bit  = 0x00;
char check_bit      = start_bit ^ cmd_num_bit ^ temp_num1_bit ^ temp_num2_bit;
```

Your ESP32 receiver needs to parse **5 raw bytes**, not a text line
terminated by `\n`.

### 3.3 Open question — does the cloud SDK let you emit custom ASCII instead?

The platform tutorials describe configuring wake words / command words /
replies, but the retrievable documentation doesn't show whether the
**cloud-generated SDK** lets you override the wire format to plain ASCII
strings, or whether it always emits the byte-code format above with
command numbers you assign on the platform.

**Do this before writing any ESP32 parsing code:** build a minimal
product on the platform with just 1–2 test commands, flash it, and
capture the raw bytes on a logic analyzer or a plain serial terminal set
to show hex. Confirm the actual wire format your specific build emits,
then write the ESP32 parser to match. Don't assume ASCII until you've
seen it on the wire.

### 3.4 Reference ESP32 receiver (byte-code version — verified format)

```cpp
// VIEON — ESP32 UART2 receiver for VC-02, factory byte-code protocol
#define VC02_RX_PIN 16   // ESP32 RX2 ← VC-02 TX1
#define VC02_TX_PIN 17   // ESP32 TX2 → VC-02 RX1
HardwareSerial VC02(2);

void setup() {
  Serial.begin(115200);
  VC02.begin(115200, SERIAL_8N1, VC02_RX_PIN, VC02_TX_PIN);
  Serial.println("VIEON: VC-02 bridge online.");
}

void loop() {
  static uint8_t frame[5];
  static uint8_t idx = 0;

  while (VC02.available()) {
    uint8_t b = VC02.read();
    if (idx == 0 && b != 0x5A) continue;   // wait for start bit
    frame[idx++] = b;
    if (idx == 5) {
      uint8_t chk = frame[0] ^ frame[1] ^ frame[2] ^ frame[3];
      if (chk == frame[4]) {
        Serial.printf("VC-02 command number: 0x%02X\n", frame[1]);
        // dispatch based on frame[1] here
      } else {
        Serial.println("VC-02: checksum mismatch, dropping frame");
      }
      idx = 0;
    }
  }
}
```

---

## 4. Flashing Over UART

Confirmed from AI-Thinker's own documentation:

- Wiring for flashing is the **same as the normal communication
  wiring** (TX1/RX1) — no special reconfiguration needed.
- Tool: **`UniOneUpdateTool.exe`**
- Firmware: **`uni_app_release_update.bin`** — must be the `update`
  variant, not the JTAG-only full image.
- The tool's port list highlights a port with a **yellow background**
  once it's successfully opened — that's your confirmation the port is
  live.

**⚠️ UNVERIFIED — treat with caution, get a second source before
relying on it:**
- Specific flashing baud rate (460800 / 921600) — not confirmed from
  official docs in this pass.
- Any BOOT/RST button-hold sequence — AI-Thinker's own doc doesn't
  describe one for the standard serial-burn path (it explicitly says
  "no special precautions" beyond matching the communication wiring).
  If your tool hangs on "Connecting," check the tool's own help/log
  output rather than assuming a specific button combo — a wrong
  sequence risks nothing electrically, but wastes time chasing a
  fabricated procedure.

---

## 5. Power & Wiring (confirmed against datasheet)

- VC-02 needs **5V supply, current capability ≥ 500 mA** (peak draw
  during wake-word inference).
- **TX1/RX1 are 3.3V logic** — safe to wire directly to ESP32 GPIO
  (also 3.3V logic), no level shifting needed.
- **Do not power the VC-02 from the ESP32's 3.3V regulator pin** — that
  regulator can't supply the current the VC-02 needs under load. Give
  it its own 5V rail (bench supply, MB102 board, or a dedicated 5V
  buck converter), sharing a **common ground** with the ESP32.

```
VC-02                         ESP32 DevKit
-----                         ------------
5V   ── external 5V rail (NOT ESP32 3V3 pin)
GND  ── common ground ─────── GND
TX1  ─────────────────────── GPIO16 (RX2)
RX1  ─────────────────────── GPIO17 (TX2)
```

Standard breadboard hygiene that applies here regardless of module:
common star ground, a bulk capacitor (a few hundred µF) across the 5V
rail near the VC-02 to absorb its current transients, and don't run the
VC-02 and ESP32 off the same USB port's 500mA limit.

---

## 6. Verified Sources

- AI-Thinker Offline Voice Module docs — https://docs.ai-thinker.com/en/voice_module
- AI-Thinker product page (US516P6 confirmed) — https://en.ai-thinker.com/pro_view-101.html
- Tara (AI-Thinker) — VC-02 Tutorial 1: Creating a product — Medium
- Tara (AI-Thinker) — VC-02 Tutorial 2: Building the SDK — Medium
- Tara (AI-Thinker) — VC-02 Tutorial 3: Re-editing product function — Medium
- AI-Thinker (official account) — "Is the SDK Open Source?" — Hackster.io, Nov 27 2024 (source for the byte-code protocol and burning tools)
- VC-02 datasheet (Mikroe mirror) — power/voltage specs
- Arduino Forum — "How to program AI-Thinker VC-02" (community troubleshooting thread, useful for context but not treated as authoritative above)

---

## 7. What to Verify Yourself Before Trusting This Fully

1. Capture the raw UART bytes from your specific flashed build and
   confirm it matches §3.2's format before writing production parsing
   code.
2. Confirm your exact flashing baud rate by checking `UniOneUpdateTool.exe`'s
   own defaults/help — don't assume 921600 or 460800.
3. If the tool doesn't connect, check its own log/status text rather
   than trying an unconfirmed button sequence.
