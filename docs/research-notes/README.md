# Research Notes

Background notes collected while designing Padma SOS, written as lecture notes. They explain *why* the selection docs in [`../architecture/`](../architecture/README.md) look the way they do.

These notes are **background, not decisions**. Decisions live in the architecture and requirements docs. Numbers from memory are marked *(verify)*; check them against the datasheet before relying on them.

## Contents

1. [Module vs chipset](#1-module-vs-chipset)
2. [Combined LTE + GNSS module, or two modules?](#2-combined-lte--gnss-module-or-two-modules)
3. [GNSS basics: constellations vs bands](#3-gnss-basics-constellations-vs-bands)
4. [L1-only vs L1 + L5 modes](#4-l1-only-vs-l1--l5-modes)
5. [Standard vs high-precision GNSS](#5-standard-vs-high-precision-gnss)
6. [u-blox GNSS form factors](#6-u-blox-gnss-form-factors)
7. [VoLTE, IMS and emergency calls](#7-volte-ims-and-emergency-calls)
8. [APN (Access Point Name)](#8-apn-access-point-name)
9. [VoLTE call vs SMS for an SOS](#9-volte-call-vs-sms-for-an-sos)
10. [Radio certifications (CE, FCC, India)](#10-radio-certifications-ce-fcc-india)
11. [LTE Cat 1 vs Cat 1 bis, and why VoLTE depends on the chipset](#11-lte-cat-1-vs-cat-1-bis-and-why-volte-depends-on-the-chipset)
12. [Does the MCU need an NPU?](#12-does-the-mcu-need-an-npu)
13. [Open-source hardware: licenses and OSHWA certification](#13-open-source-hardware-licenses-and-oshwa-certification)
14. [Testing prototypes: where, and what not to do](#14-testing-prototypes-where-and-what-not-to-do)

---

## 1. Module vs chipset

- A **chipset** is the silicon: the modem chip from Qualcomm, UNISOC, ASR, Nordic, etc.
- A **module** is a small, pre-certified board built around a chipset. It adds the RF front end, power circuits and firmware, and the vendor sells it as one solderable part (Quectel, Telit, Cavli, u-blox …).

The **module vendor and the chipset vendor can be different companies, and even different countries.** For example, the Quectel EG915U uses a UNISOC chipset (believed to be the 8910DM), and the Cavli C16QS uses a Qualcomm QCX216.

Some modules also have an **application processor** that can run your own code ("OpenCPU" / SDK). Examples: the Cavli P16QS (RISC-V at 325 MHz) and the C17QS (dual Cortex-M3 at 306 MHz).

## 2. Combined LTE + GNSS module, or two modules?

A combined module looks smaller, but for Padma SOS **two separate modules are better**:

| Point | Combined module | Separate LTE + GNSS |
|---|---|---|
| NavIC L5 | Integrated GNSS is usually **L1-only**, so no NavIC | Choose any dual-band L1 + L5 receiver |
| GNSS during a call | GNSS often shares RF / baseband with LTE and can degrade or pause during transmission, which is exactly when we need location (SOS call) | Independent |
| Power | Every location fix wakes the whole modem | The MCU can run GNSS while the modem sleeps in PSM |
| Size saving | Small. You still need a GNSS antenna and RF layout | The LC79H is only ~10 × 10 mm |
| Choice of parts | Locked to modules that combine both | Best LTE part + best GNSS part |

Combined modules are a good fit for large, mains-powered trackers. They are a poor fit for a tiny VoLTE wearable.

## 3. GNSS basics: constellations vs bands

These are two **independent** ideas.

**Constellation = which satellite system:**

| Constellation | Operator | Coverage |
|---|---|---|
| GPS | USA | Global |
| Galileo | EU | Global |
| GLONASS | Russia | Global |
| BeiDou | China | Global |
| **NavIC** (IRNSS) | India | India + ~1500 km around it |
| QZSS | Japan | Asia-Pacific |

**Multi-constellation** gives more satellites in view, which means faster and more reliable fixes between buildings or with the body blocking the sky.

**Band = which radio frequency:**

- **L1 (~1575 MHz):** the classic band that every receiver uses.
- **L5 (~1176 MHz):** a newer, stronger, wider signal.

**Multi-band (L1 + L5)** improves accuracy in two ways:
- The ionosphere delays the two frequencies by different amounts, so comparing them cancels most of that error.
- L5 copes much better with reflections off buildings. In the open it gives roughly 1–2 m instead of 2–5 m, and the gain is much bigger in cities.

**Together** (rows = constellations, columns = bands):

| | L1 band | L5 band |
|---|---|---|
| GPS | L1 C/A | L5 |
| Galileo | E1 | E5a |
| BeiDou | B1 | B2a |
| **NavIC** | only on the newest (NVS) satellites, few today | **L5** (satellites in service) |

**Key point:** NavIC broadcasts mainly on **L5**, so an L1-only receiver can't use NavIC today, however many constellations it lists. That is why gate G1 needs dual-band.

## 4. L1-only vs L1 + L5 modes

| Mode | Usually available? | Notes |
|---|---|---|
| L1 + L5 | Yes, the default on dual-band parts | Best accuracy, includes NavIC |
| L1 only | Usually configurable | Saves some current (the second RF chain is off) |
| L5 only | Rare | Receivers normally acquire on L1 and add L5 |

How it is configured:
- **u-blox F10:** configuration keys switch individual signals on and off, per constellation and per band.
- **Quectel LC29H / LC79H (Airoha chip):** proprietary commands; check the protocol manual for a separate L5 switch.

**Possible use:** L1-only for idle breadcrumbs and L1 + L5 during an SOS. The trade-off is that L1-only loses NavIC. Decide once the current saving is known; it may only be a few mA.

## 5. Standard vs high-precision GNSS

- **High precision** means RTK / multi-band (L1/L2/L5) receivers with centimetre-level accuracy. They need a correction data stream and a large, good antenna, and they use much more power.
- **Our target** (NFR-021) is **≤ 10 m CEP50** (50 % of fixes within 10 m).
- **Standard dual-band L1 + L5** receivers already claim about 1 m CEP, 10× better than needed.

In a body-worn device, **the antenna and body shadowing limit accuracy, not the receiver grade.** Effort on antenna placement and on isolating the receiver from the LTE transmitter pays off more than a high-precision receiver.

## 6. u-blox GNSS form factors

Sorted by footprint. The sizes are typical values from memory *(verify)*; they vary by generation (M8 / M10 / F10).

| Form factor | Typical size (mm) | Area (mm²) | Built-in antenna? |
|---|---|---|---|
| **MIA** | 4.5 × 4.5 × 1.0 | ~20 | No |
| **EVA** | 7.0 × 7.0 × 1.1 | ~49 | No |
| **MAX** | 9.7 × 10.1 × 2.5 | ~98 | No |
| **CAM** | 9.6 × 14.0 × 1.95 | ~134 | Yes (chip antenna) |
| **DAN** | ~10.5 × 16.0 × 3 | ~168 | Yes (L1 / L5 antenna) |
| **NEO** | 12.2 × 16.0 × 2.4 | ~195 | No |
| **SAM** | 15.5 × 15.5 × 6.3 | ~240 | Yes (patch antenna) |
| **LEA** | 17.0 × 22.4 × 2.4 | ~381 | No |

- **Compare like with like:** CAM, DAN and SAM include the antenna; the others still need one, plus clearance around it.
- **Platforms:** **M10** is L1-only with no NavIC (e.g. MAX-M10S and MIA-M10Q fail G1). **F10** is dual-band L1 + L5 with NavIC L5 (NEO-F10N, DAN-F10N, and possibly MAX-F10S *(verify)*).

## 7. VoLTE, IMS and emergency calls

- On LTE there is no classic circuit-switched voice. **All calls go over IMS**, the operator's voice-over-IP system; that is what **VoLTE** means.
- **An IMS emergency call is a separate procedure** from a normal VoLTE call. A module can support normal VoLTE and still handle 112 badly.

| | Normal VoLTE call | IMS emergency call |
|---|---|---|
| Number | Dialled as usual | The module must recognise 112 as an emergency number (from its own list, the SIM's list and the network's list) |
| Connection | Normal `ims` APN | Separate **emergency connection** (`sos` APN) with **highest priority** |
| Registration | Must be registered with its SIM | **Emergency registration**, or in some cases none |
| Address | A phone number | `urn:service:sos` |
| Network | Own operator only | **Any** available operator |

**3GPP history:**
- IMS emergency calls: Release 7 (TS 23.167).
- LTE emergency attach and emergency bearers: **Release 9** (TS 23.401, TS 24.301).
- Supporting a release doesn't mean a module implements every optional feature, so ask the vendor.

**Emergency connection vs emergency attach:**

| Situation | What happens |
|---|---|
| SIM works, own operator available | Normal attach (already done) + **emergency connection** for the 112 call |
| No SIM, rejected SIM, or only another operator's coverage | **Emergency attach**: the network lets the device in for emergency calls only |

- **The emergency connection** is what gives the 112 call priority. It is used in every 112 call.
- **The emergency attach** is the fallback way in. It only works if the network allows it: each cell announces whether it does, and many LTE operators disable SIM-less emergency calls. On 2G it almost always worked.
- **The valuable case for us:** calling 112 over another operator's network when the wearer's own operator has no coverage.

## 8. APN (Access Point Name)

An APN is the **name of a "door" into the operator's network**. The device states it when opening a data connection, and the operator routes the traffic accordingly. A phone typically has several connections open at once:

| APN | Used for |
|---|---|
| `jionet`, `airtelgprs.com`, … | Internet data |
| `ims` | VoLTE calls, SMS over IMS |
| `sos` | Emergency calls (priority) |
| private (e.g. `company.m2m`) | IoT traffic to a company's private network |

- **The module normally picks these automatically** from the SIM and the operator's settings.
- **With IoT / M2M SIMs**, the operator may give a private APN and may **restrict voice**. Check this when choosing a SIM plan, because the device must be able to call 112.

## 9. VoLTE call vs SMS for an SOS

| | VoLTE call to 112 | SMS with location |
|---|---|---|
| Live audio from surroundings | ✅ | ❌ |
| Priority / speed | Emergency connection, real-time | Best effort, can be delayed |
| Weak signal | Can drop | Small, retried automatically, so often gets through |
| Hardware | Needs VoLTE + digital audio | Works on data-only modules too |

**Decision:** VoLTE stays a hard gate. SMS to 112 runs **in parallel** (FR-025).

**Padma SOS calls are one-way.** There is no speaker (FR-010) and the downlink is muted (FR-022). So:
- The announcement must be clear and immediate, and the location repeats during the call (FR-023), because operators may hang up on silent calls (OQ-14).
- If 112 calls back, the device auto-answers silently (FR-028, OQ-17).

**Forensic audio** (a recording for evidence) is a different thing from live audio to 112. It could go over data, but it raises consent and legal questions. FR-004 currently forbids storing audio.

## 10. Radio certifications (CE, FCC, India)

A certification proves a radio product:
- uses its frequencies correctly,
- doesn't disturb other equipment,
- is electrically safe,
- keeps radio exposure to the body within limits.

**Each country has its own scheme.**

**CE (EU)** is the manufacturer's declaration of conformity. For radios, the law is the **Radio Equipment Directive (RED)**, which covers:
- **Article 3.2, spectrum:** transmits only in its bands, at the right power. The LTE test standard is EN 301 908.
- **Article 3.1b, EMC:** doesn't disturb other electronics, and isn't upset by them.
- **Article 3.1a, safety and health:** electrical safety and RF exposure.

**Other logos you'll see:**

| Logo | Meaning | Legal? |
|---|---|---|
| FCC / IC | USA / Canada approval | Yes |
| UKCA | UK version of CE | Yes |
| GCF / PTCRB | Industry tests of 3GPP behaviour | No (industry) |
| Carrier (Verizon, Orange, …) | Tested on that operator's network | No (commercial) |

**None of these count in India.**

**Module certificate vs product certificate:**
- The module vendor certifies the module.
- We still certify the **finished device**.
- A certified module lets the radio tests reuse the module's reports. EMC, safety and SAR are always tested on the whole device.

**India:**

| Certification | Authority | Covers |
|---|---|---|
| **TEC MTCTE** | DoT / TEC | Telecom equipment: EMC, safety, radio, **security**. Cellular smartwatches and trackers with embedded SIMs are on the list; enforcement dates keep being postponed |
| **WPC ETA** | DoT, Wireless Planning & Coordination wing | Permission to sell or import radio products; needed for customs |
| **BIS CRS** | Bureau of Indian Standards | Safety of listed electronics, **Li-ion cells / batteries** (IS 16046), chargers |
| **SAR** | DoT | Radio energy absorbed by the body; matters for a body-worn LTE device |
| **Jio / Airtel approval** | Operators (commercial) | Works on their network, especially **VoLTE / 112** |

**Ask module vendors for:** MTCTE and WPC ETA for the exact variant, Jio / Airtel VoLTE and 112 testing, and reusable test reports. **Before certifying the product**, confirm the current rules with a TEC-designated lab, because they change often.

## 11. LTE Cat 1 vs Cat 1 bis, and why VoLTE depends on the chipset

**The difference is the number of receive antennas:**

| | Cat 1 | Cat 1 bis |
|---|---|---|
| 3GPP release | Release 8 | **Release 13** |
| Receive antennas | **2** (main + diversity) | **1** |
| Peak speed | ~10 Mbit/s down / 5 up | Same |
| Typical use | Older IoT, POS terminals, routers | Wearables, trackers, small IoT |

**Diversity** means two antennas that receive the same signal. The modem combines them, which helps when one antenna is faded or blocked. Cat 1 **requires** it. Cat 1 bis was created so that small devices can drop the second antenna, at the cost of slightly worse reception.

**Antenna count for Padma SOS:**

| Option | LTE antennas | GNSS antenna | Total |
|---|---|---|---|
| Cat 1 bis | 1 | 1 | **2** |
| Cat 1 | 2 | 1 | **3** |

In a 46 × 39 mm wearable, every antenna needs space, clearance and tuning, and the body detunes them. Three antennas is very hard. Some Cat 1 modules can switch the diversity path off with a setting, but reception gets worse and operator acceptance can suffer. That is why K8 asks for Cat 1 bis.

**"Cat 1 bis" says nothing about voice.** Whether a module has VoLTE depends mostly on the **chipset and the vendor's firmware**:

| Chipset family | Designed for | VoLTE? | Example modules |
|---|---|---|---|
| UNISOC 8910DM | Low-cost feature phones (China) | ✅ Built in | Quectel EG915U, EC200U |
| Qualcomm QCX216 | IoT data (meters, trackers) | ❌ Usually left out | Cavli C16QS / CQ16 *(chipset: verify)* |
| Nordic, Sequans, Trasna LEXI / ADAM | IoT data | ❌ | nRF93M1, LEXI-R10 |

**Example:** Cavli confirmed (2026-11-24) that **none of its Cat 1 bis modules support VoLTE, only its Cat 1 modules**. Other vendors (Quectel, Telit, Trasna LENA-R8) do offer Cat 1 bis with VoLTE.

**Lesson:** "VoLTE support" on a web page is not proof. Check the spec sheet, and ask the vendor in writing.

## 12. Does the MCU need an NPU?

**An NPU (neural processing unit)** is a small hardware accelerator for neural networks, such as Arm's **Ethos-U55**. It does the multiply-accumulate work of a model much faster, and with **much less energy per inference**, than the CPU.

**Not strictly required. It depends on what has to run *always on*.** Split the audio workload into two parts:

| Task | When it runs | Typical model size | Can a plain Cortex-M4F / M33 do it? |
|---|---|---|---|
| **Keyword spotting** (e.g. a code word) | **Always on** | Tens of kB, a few M-ops per inference *(verify)* | ✅ Yes, the classic MCU workload |
| **Scream / sound-event detection** | Always on | Similar to keyword spotting | ✅ Usually |
| **Speaker verification** (is it the wearer?) | Only **after** a trigger | Hundreds of kB to MB, 10–100× more compute *(verify)* | ⚠ Possible but slow; latency and SRAM are the risk |

**Why the "always on" part decides it:**
- **Battery life** is set by the always-on current, not by the occasional burst. An NPU runs the same model at a fraction of the energy, so it makes always-on listening cheaper.
- **Burst work**, such as speaker verification after a keyword, only costs **latency**. Taking 1 s instead of 0.1 s on an M33 may be acceptable *(verify by benchmark)*.

**Cortex-M55 with Helium.** Helium is the M55's vector (SIMD) extension. It speeds up DSP and ML several times over an M4, even **without** an NPU *(verify factor)*. An M55 + Ethos-U55 part (e.g. **Alif Ensemble E1 / E3**) gives both.

**When an NPU becomes necessary:**
- Speaker verification must run **continuously**, or must be **fast**
- Models grow (better accuracy, more keywords, noise robustness)
- The always-on current budget is too tight for the CPU alone

**Recommended way to decide (OQ-10 / OQ-15):** benchmark the same keyword and speaker models on two dev kits, measuring:
- always-on current (µA)
- latency
- SRAM use

| Kit | Class |
|---|---|
| Cortex-M33 (e.g. STM32U5, nRF54) | No NPU |
| Alif Ensemble E1 / E3 | M55 + Ethos-U55 |

## 13. Open-source hardware: licenses and OSHWA certification

**Open-source hardware (OSHW)** means anyone may study, modify, make **and sell** the design. The definition comes from the **Open Source Hardware Association (OSHWA)**.

**What has to be published, and under which license:**

| Part | What to publish | Typical licenses |
|---|---|---|
| Hardware | **Editable source files** (KiCad project, not just PDFs or Gerbers), bill of materials, enclosure CAD | CERN-OHL (-P / -W / -S), CC-BY, CC-BY-SA |
| Firmware you write | Source code | MIT, BSD, Apache-2.0, GPL |
| Documentation | Build and assembly instructions | CC-BY, CC-BY-SA |

**"Non-commercial" licenses (CC-BY-NC) are not open source hardware.** Selling must be allowed.

**OSHWA certification:**
- **Free, self-certified online.** Nobody inspects or tests the product.
- You get a **unique ID** (country code + number, e.g. `NL000123`), the right to use the **OSHW logo**, and a listing in the public directory.
- **It is a promise.** OSHWA can withdraw the label if the design isn't actually open.
- **It is not a safety or regulatory certification.** CE, TEC MTCTE, WPC and the rest are still needed (section 10).
- **Bought parts may be closed.** The LTE module and its firmware are proprietary, and that is allowed. Certification covers what *you* design.

**Hybrid model: open hardware, closed product.**
- The board and enclosure are released openly. The firmware, app and services stay closed.
- This is legitimate, and many companies do it, **but it cannot be OSHWA certified**, because certification needs the firmware you write to be open as well.

**Licensing traps while undecided:**
- **Publishing is one-way.** A version released under an open license can't be taken back. Keep the repo private until the decision is made.
- **GPL is "copyleft".** GPL code linked into your firmware forces the firmware to be GPL. Prefer MIT, BSD or Apache libraries (Zephyr and most vendor SDKs are fine) to keep both options open.
- **CERN-OHL variants:** -P (permissive), -W (weakly reciprocal), -S (strongly reciprocal, like GPL for hardware).

## 14. Testing prototypes: where, and what not to do

**Where, by kind of work:**

| Work | Typical place | Equipment needed |
|---|---|---|
| Soldering, rework, quick fixes | Hackerspace (e.g. **RevSpace**, Den Haag / Leidschendam) or own bench | Soldering station, hot air, microscope, fume extraction |
| Electrical measurements | Company lab | Oscilloscope, bench supply, logic analyser |
| **Current profiling** (sleep µA to call peaks) | Company lab | Nordic PPK2, Qoitech Otii, or a source-measure unit |
| RF and antenna tuning | Company lab or university (TU Delft) | Spectrum analyser, **VNA** (vector network analyser) |
| Pre-compliance (EMC, emissions) | University or test house | Near-field probes, EMC receiver, shielded room |
| Network and call tests | Test house or operator lab | **Network simulator** (call box), e.g. R&S CMW / Anritsu |

**Hackerspaces are good for building and for advice. They rarely have RF equipment.**

**Never make real 112 test calls on a live network.**
- Test calls tie up real emergency dispatchers, and in most countries that is an offence or at least unwelcome.
- Test the call flow with an **ordinary phone number**.
- Test the **IMS emergency procedure** (section 7) on a **network simulator**, which can pretend to be an operator, including the emergency connection, or in a test arranged with the operator or ERSS.

**Check that the bands overlap before buying a local SIM.**
- The modules are chosen for **Indian** bands (B1 / B3 / B5 / B8 / B40 / B41).
- The Netherlands mainly uses B1 / B3 / B7 / B8 / B20 / B28 *(verify per operator)*.
- The overlap (B1, B3, B8) is enough for bench tests in Europe. Confirm against the module's band list first.
