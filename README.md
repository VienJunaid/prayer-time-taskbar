# 🕌 Prayer Time Taskbar — Windhawk Mod for Windows 11

A Windhawk mod that adds a small prayer indicator next to the Windows 11 taskbar clock (bottom right). It shows:

- **An icon** (🕌)
- **The current prayer period** you're in (e.g. `Asr`)
- **A countdown** (hours:minutes) until that prayer's time ends

```
🕌 Asr · 1:23 left        11:42 PM
                          11/3/2026
```

All prayer times are **calculated on your PC using the ISNA method**. No internet, no API, no account.

---

## ✨ Features

| Feature | Status |
|---|---|
| Icon + current prayer + time left, next to the taskbar clock | Planned (v0.1) |
| Offline ISNA calculation (Fajr 15°, Isha 15°) | Planned (v0.1) |
| Shafi'i / Hanafi Asr option | Planned (v0.1) |
| Location and offsets set in Windhawk's Settings tab | Planned (v0.1) |
| Automatic daylight saving time handling | Planned (v0.1) |
| Warning style when time is almost up (e.g. < 15 min) | Planned (v0.2) |
| Hover tooltip showing all 5 prayer times for today | Planned (v0.2) |
| Notification at the start of each prayer | Idea |
| Custom icon image instead of an emoji | Idea |

---

## 🕰️ Prayer Periods (what "time left" means)

The countdown is **how much time remains to pray the current prayer**, meaning time until that prayer's window ends.

| Current period | Starts at | Ends at (countdown target) | Display example |
|---|---|---|---|
| **Fajr** | Fajr | Sunrise | `🕌 Fajr · 0:47 left` |
| *(No prayer)* | Sunrise | Dhuhr | `🕌 Dhuhr in 3:10` |
| **Dhuhr** | Dhuhr | Asr | `🕌 Dhuhr · 2:05 left` |
| **Asr** | Asr | Maghrib (sunset) | `🕌 Asr · 1:23 left` |
| **Maghrib** | Maghrib | Isha | `🕌 Maghrib · 0:58 left` |
| **Isha** | Isha | Next Fajr *(default)* or Islamic midnight *(option)* | `🕌 Isha · 6:12 left` |

**Notes**
- Between **Sunrise and Dhuhr** there's no obligatory prayer, so the mod shows a countdown *to* Dhuhr instead.
- **Isha end** is a setting:
  - `fajr` (default): Isha lasts until the next day's Fajr.
  - `midnight`: Isha ends at Islamic midnight, halfway between Maghrib and the next Fajr.
- The timer counts down every minute. It shows `H:MM`, rounding down so it never shows more time than you actually have.

---

## ⚙️ Settings (Windhawk Settings tab)

| Setting | Type | Default | Description |
|---|---|---|---|
| `latitude` | string | `33.7490` | Your latitude (decimal degrees, + = North) |
| `longitude` | string | `-84.3880` | Your longitude (decimal degrees, − = West) |
| `asrMethod` | `shafii` / `hanafi` | `shafii` | Shadow factor 1 (Shafi'i, Maliki, Hanbali) or 2 (Hanafi) |
| `ishaEnd` | `fajr` / `midnight` | `fajr` | When the Isha countdown ends |
| `icon` | string | `🕌` | Any emoji or character (e.g. `☪`, `🌙`) |
| `format` | string | `{icon} {prayer} · {time} left` | Display template |
| `formatNoPrayer` | string | `{icon} {next} in {time}` | Template for the Sunrise→Dhuhr gap |
| `offsetFajr` … `offsetIsha` | int (minutes) | `0` | Fine-tune each prayer to match your local mosque |
| `warningMinutes` | int | `15` | (v0.2) Highlight when this little time is left |

Numbers with decimals are stored as strings because Windhawk settings only support integer numbers.

---

## 🧮 How the Times Are Calculated (Offline)

**Simple version:** prayer times depend on where the sun is in the sky. Given your latitude, longitude and the date, we can calculate exactly when the sun reaches each position.

**Analogy:** it's like knowing a train's exact route and speed. You don't need to watch the station board, because you can work out the arrival times yourself.

### ISNA parameters
| Prayer | Rule |
|---|---|
| Fajr | Sun **15°** below the horizon (before sunrise) |
| Sunrise | Sun's top edge at the horizon (**0.833°** below, which accounts for refraction) |
| Dhuhr | Sun crosses the meridian (solar noon) |
| Asr | Object's shadow = shadow at noon + **1×** its length (Shafi'i) or **2×** (Hanafi) |
| Maghrib | Sunset (same 0.833° as sunrise) |
| Isha | Sun **15°** below the horizon (after sunset) |

### Algorithm (per day)
1. **Days since J2000:** `D = JulianDate − 2451545.0`
2. **Sun position:**
   - `g = 357.529 + 0.98560028·D` (mean anomaly)
   - `q = 280.459 + 0.98564736·D` (mean longitude)
   - `L = q + 1.915·sin(g) + 0.020·sin(2g)` (ecliptic longitude)
   - `e = 23.439 − 0.00000036·D` (obliquity)
   - `RA = atan2(cos(e)·sin(L), cos(L)) / 15` (right ascension, hours)
   - `δ = asin(sin(e)·sin(L))` (declination)
   - `EqT = q/15 − RA` (equation of time, normalized to ±12h)
3. **Solar noon (UTC):** `Dhuhr = 12 − longitude/15 − EqT`
4. **Time from noon until the sun reaches angle α below the horizon:**
   `T(α) = (1/15) · acos( (−sin α − sin φ · sin δ) / (cos φ · cos δ) )`
5. **Asr (shadow factor t = 1 or 2):**
   `A(t) = (1/15) · acos( (sin(acot(t + tan|φ − δ|)) − sin φ · sin δ) / (cos φ · cos δ) )`
6. **Results (UTC):**
   - Fajr = Dhuhr − T(15)
   - Sunrise = Dhuhr − T(0.833)
   - Asr = Dhuhr + A(t)
   - Maghrib = Dhuhr + T(0.833)
   - Isha = Dhuhr + T(15)
7. **Convert UTC to local time** with the Windows API (`SystemTimeToTzSpecificLocalTime`) so **daylight saving time is automatic**.
8. **Apply the per-prayer minute offsets** from settings.

Here φ is latitude and all trig is in degrees. Formulas follow the well-known PrayTimes.org method; this project uses an independent implementation.

**High latitudes:** if `acos` gets a value outside [−1, 1], meaning the sun never reaches 15° below the horizon (summer in far-north locations), fall back to the "angle-based" rule: night portion = 15/60 of the night. This doesn't happen in the continental US but keeps the mod from crashing.

**Recalculate** once at startup, at local midnight, and whenever settings change or the system time or time zone changes.

---

## 🏗️ Architecture

```
explorer.exe (Windows 11 taskbar)
 └── Windhawk injects mod
      ├── PrayerCalc     → today's + tomorrow's times (pure math, no I/O)
      ├── PeriodLogic    → "which period am I in?" + minutes left
      ├── Formatter      → builds "🕌 Asr · 1:23 left" from the template
      └── ClockHook      → hooks the taskbar clock's text update and adds our line
```

- **Target process:** `explorer.exe` (x86-64)
- **Clock hook:** use the same approach as the existing Windhawk mod **Taskbar Clock Customization** (by m417z), which hooks the Windows 11 clock's text-update functions. Use its source as the reference for hook targets and symbols.
- **Update rate:** the clock already refreshes every minute, which is enough for an `H:MM` countdown. No extra timer is needed for v0.1.
- **Icon:** v0.1 uses an emoji or character in the text (simplest and most reliable). A real image icon via XAML injection is a later idea.

---

## 📁 Repository Structure

```
prayer-time-taskbar/
├── README.md                     ← this file
├── LICENSE
├── mod/
│   └── prayer-time-taskbar.wh.cpp   ← the Windhawk mod (single file)
├── tools/
│   └── verify_times.py           ← optional: prints a year of times to compare against a timetable
└── docs/
    └── screenshots/
```

Windhawk mods are a **single `.wh.cpp` file** that contains the code, the metadata header, the readme and the settings schema.

---

## 🚀 Installation (for users)

1. Install **Windhawk** from [windhawk.net](https://windhawk.net).
2. Open Windhawk and click **Create a New Mod**.
3. Replace the template with the contents of `mod/prayer-time-taskbar.wh.cpp`.
4. Click **Compile Mod**, then **Exit Editing Mode**.
5. Open the mod's **Settings** tab, then enter your latitude and longitude and choose your Asr method.
6. The prayer indicator appears next to your clock.

---

## ✅ Testing Checklist

- [ ] Times match a trusted ISNA timetable (e.g. your mosque or an ISNA-based app) within ±1–2 minutes.
- [ ] Each period switch happens at the right moment: Fajr → gap → Dhuhr → Asr → Maghrib → Isha → Fajr.
- [ ] Isha countdown crosses midnight correctly (uses **tomorrow's** Fajr).
- [ ] DST change days (2nd Sunday of March, 1st Sunday of November) are correct.
- [ ] Hanafi setting makes Asr later.
- [ ] Changing settings updates the display without a reboot.
- [ ] Works with the taskbar on multiple monitors.
- [ ] Disabling the mod restores the original clock.
- [ ] Works fully with Wi-Fi turned off.

---

## 🗺️ Roadmap

- **v0.1:** Icon, current period, and countdown. ISNA offline calculation with settings.
- **v0.2:** Warning color or text when under `warningMinutes`. Hover tooltip with all 5 times.
- **v0.3:** Optional toast notification at each prayer start.
- **Later:** More methods (MWL, Umm al-Qura, Egyptian), Hijri date, custom image icon.

---

## 📜 License

MIT (suggested). Pick your license when creating the repo.

## 🙏 Credits

- [Windhawk](https://windhawk.net) by Ramen Software
- Taskbar Clock Customization mod (reference for clock hooking)
- Astronomical formulas based on the PrayTimes.org method
