# 🦯 Smart Headband with VL53L1X Sensors

**9 months of development. 4 ToF sensors. Works indoor. Fails outdoor.**

This repository documents what I learned about the real-world limitations of VL53L1X sensors in outdoor environments.

## 🎬 Project video

[→ Watch the full reel](https://www.instagram.com/reel/DTgGp6-iFGg/)

## ⚡ Summary

I designed a wearable headband with 4 laser distance sensors (VL53L1X) to assist visually impaired people in detecting obstacles.

**Result:**
- ✅ **Indoor**: perfect performance (0.2–1.2 m, stable feedback)
- ❌ **Outdoor**: 80–95% invalid readings, even under overcast sky

**Cause:** the VL53L1X uses a 940 nm laser. The sun emits massive radiation at exactly 940 nm, saturating the receiver. No optical filter solves this: it's a physical limitation of the component.

## 🔧 Hardware used

- ESP32 WROOM-32
- 4× VL53L1X (Adafruit breakout)
- I²C multiplexer TCA9548A
- 4× 3V active buzzers (acoustic feedback)
- Gilisymo certified optical covers (VL53L1X-specific)

**Software:** Pololu VL53L1X library (better firmware control than Adafruit)

### The prototype

![Headband front view](images/headband_overview_front.jpg)
*Front view of the headband with electronics and mounted sensors*

![Sensor detail](images/sensors_detail_front.jpg)
*Detail of the 4 VL53L1X sensors with Gilisymo optical covers and protective shields*

![Top view](images/headband_overview_above.jpg)
*Top view: sensor layout and system architecture*

## 🧪 What I tried

1. **Generic IR filters** → excessive crosstalk
2. **Certified optical covers** → indoor improvement, outdoor insufficient
3. **Mechanical shielding** → partial noise reduction, problem persists
4. **Software filters** (debounce, moving average) → can't fix missing valid data

## 📊 Real-world data

Invalid measurement percentage under different conditions:

| Condition | Valid readings | Reliable range |
|-----------|---------------|-----------------|
| Indoor | >95% | 1.3 m |
| Overcast sky | 10–20% | 0.8 m |
| Direct sunlight | <5% | 0.3 m |

With <20% valid readings, acoustic feedback becomes irregular and unreliable.

## 💡 Conclusion for makers

If you need outdoor obstacle detection, the VL53L1X is **not the right choice**.

**Recommended alternatives:**
- **Ultrasonic** (JSN-SR04T): affordable, waterproof, 4.5m range
- **Industrial ToF** (Benewake TFmini-S Plus): outdoor-rated laser, 12m range
- **mmWave radar 60 GHz**: immune to light, rain, fog

## 📄 Full technical documentation

Want all the details on SNR, firmware configurations, alternative analysis?

→ [Complete technical report (ITA)](docs/vl53l1x_outdoor_analysis.md)  
→ [Full technical analysis (ENG)](docs/vl53l1x_outdoor_analysis_en.md) *(coming soon)*

## 📜 License

MIT License - use, modify, share freely.
