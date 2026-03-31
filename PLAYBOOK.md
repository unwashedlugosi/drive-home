# How to Build a TripTik App

A complete guide to building a road trip companion app, based on the AZ→NYC drive home (March 2026). This app was used every day of a 5-day, 2,400-mile drive and was one of the most useful things I've ever built with Claude Code. Total cost: $0.

## What You Get

A single-page web app on your phone that:
- Shows your entire route broken into days and sections
- Tracks your GPS and shows a blue caret at your next upcoming stop
- Grays out everything you've already passed
- Has **audio blurbs** for interesting stops — tap Listen and hear a 20-30 second narration about whatever you're approaching (plays through CarPlay, pauses your podcast)
- Shows **"How far?"** with real driving time/distance to any stop
- Has **Apple Maps links** on everything — every hotel, restaurant, attraction, gas station
- A **"Navigate this day from here"** button that opens Apple Maps with all the day's stops as waypoints, starting from your current GPS
- Works as a home screen app on iPhone

## The Stack (All Free)

| Component | What | Cost |
|-----------|------|------|
| HTML/JS | Single file, no build step, no framework | $0 |
| Leaflet.js | Maps (CDN) | $0 |
| Edge TTS | Voice narration (Microsoft neural voices) | $0 |
| OSRM | Real driving distance/time calculations | $0 |
| Vercel | Hosting (auto-deploys from GitHub) | $0 |
| CartoDB | Map tiles | $0 |

## Step 1: Plan the Route

Before building anything, figure out:
- Start and end points
- How many days (aim for 400-650 miles/day max)
- Overnight cities
- Which interstates/highways you'll be on

**Balance the days.** Don't front-load easy days and back-load 13-hour grinds. Spread the miles evenly. The last day should be the shortest if possible.

**Be ready to change.** On this trip, the route changed 4 times — we added a St. Louis detour, then removed it, then rerouted through Kentucky and West Virginia. The app needs to be easy to rebuild.

## Step 2: Research Stops (This Is the Killer Feature)

The audio blurbs were by far the most valuable part of the app. Invest time here.

### What to Search For

| Category | What to Look For | Example Search |
|----------|-----------------|----------------|
| **History** | Civil War, Native American, civil rights, presidential, colonial | `"historical stops I-40 [state] culture history"` |
| **Weird Americana** | Roadside oddities, folk art, ghost towns, world's largest things | `"weird roadside attractions [highway] [state] atlas obscura"` |
| **Route-specific** | Route 66 landmarks, old highway culture, neon signs | `"Route 66 [state] landmarks attractions"` |
| **Food** | Vegetarian-friendly with specific addresses | `"best vegetarian restaurants [city] 2026"` |
| **Indian truck stops** | Punjabi dhabas at truck stops (authentic, cheap, vegetarian-friendly) | `"Indian food truck stop dhaba [highway]"` |
| **Shopping** | Western wear, military surplus, outlets — whatever interests you | `"authentic western wear [city]"` |
| **Nature** | National parks, caverns, natural bridges | `"caverns near I-81 Virginia"` |
| **Culture** | Museums, theaters, music venues, craft traditions | `"museums near [highway] [state]"` |

### Density Rule: Something Every 50 Miles

The GPS tracking works by finding the nearest stop. If there's a 200-mile gap with no stops, the caret gets stuck. **Make sure there's at least one stop with coordinates every 50 miles.** Even a scenic marker or a "the stretch through these hills is pretty" entry will keep the GPS working.

### Writing Good Blurbs

Each blurb should be 2-4 sentences (~300-500 characters). Rules:
- **Lead with the most surprising fact.** "About 50,000 years ago, a nickel-iron meteorite 150 feet across slammed into this spot at 26,000 miles per hour."
- **Include a specific number or detail.** Not "it's really old" but "it's been continuously inhabited since at least 1150 AD."
- **Give context.** Why should the listener care? Connect it to something bigger.
- **Keep it conversational.** These are meant to be listened to while driving, not read in a textbook.
- **Don't use apostrophes or quotes carelessly.** They break JS strings. Use `\'` for apostrophes inside single-quoted strings. Put blurbs in an array and reference by index to avoid escaping issues entirely.

### Best Blurbs from This Trip (Examples)

**Weird history:** "Two Guns sits on the site of an Apache Death Cave, where in 1878 a group of Navajo warriors reportedly trapped and suffocated 42 Apaches inside a limestone cavern by building a fire at the entrance. In the 1920s, a con man named Harry 'Two Gun' Miller built a tourist trap here, exhibiting mountain lions in pens carved from the canyon wall."

**Cultural:** "About 20% of American long-haul truck drivers are Punjabi Sikh, and a network of dhabas — roadside Indian restaurants — has sprung up at truck stops across the interstate system to serve them. These places serve home-style Punjabi food: fresh roti, dal, saag paneer, curries."

**Oddball:** "In 1985, a convicted drug smuggler parachuted out of a plane over Kentucky with 75 pounds of cocaine strapped to his body. His parachute failed and he died. Months later, a 175-pound black bear was found dead in the Georgia woods next to a duffel bag — having consumed the cocaine."

**Craft/Industry:** "Route 11 Chips has been making small-batch kettle-cooked potato chips in Middletown since 1992. They're named for the old Valley Turnpike that I-81 replaced. You can watch the chips being made through a window and sample flavors like Chesapeake Crab and Mama Zuma's Revenge habanero."

## Step 3: Build the Data Model

All route data lives in one JavaScript object:

```javascript
const ROUTE = { days: [
  { day: 1, date: 'Mon Mar 15', from: 'City A', to: 'City B',
    miles: 450, hours: '6.5',
    overnight: { name: 'Hotel Name', detail: 'Near downtown.' },
    tip: 'Leave by 7 AM to have time for stops.',
    appleMapsUrl: 'https://maps.apple.com/?saddr=Current+Location&daddr=...',
    sections: [
      { name: 'City A to Midpoint', stops: [
        { name: 'Stop Name', type: 'history', lat: 35.123, lng: -90.456,
          desc: 'Short description with address and hours.',
          blurb: 'Longer narration text for audio playback.',
          appleMap: 'https://maps.apple.com/?address=...' },
        // more stops...
      ]},
      // more sections...
    ]
  },
  // more days...
]};
```

### Critical Rules

1. **Stops MUST be in geographic order** along the route. If stop B is 50 miles east of stop A, B comes after A in the array. If they're out of order, the GPS caret breaks.

2. **Every stop needs `lat` and `lng`** for GPS tracking to work. No coordinates = invisible to GPS.

3. **Every stop needs an `appleMap` link.** Use `https://maps.apple.com/?address=...` for addresses or `https://maps.apple.com/?ll=LAT,LNG&q=Name` for coordinates.

4. **No gaps over 50 miles** between stops with coordinates.

5. **Escape apostrophes** in single-quoted strings: `it\'s`, not `it's`.

### Stop Types

`attraction` `food` `scenic` `history` `curiosity` `truckstop` `western` `surplus` `overnight`

Each gets a distinct shape and color in the legend. Colorblind-safe — uses shapes (diamond, square, circle, triangle, chevron) not just color.

## Step 4: Generate Audio

### Install Edge TTS
```bash
pip3 install edge-tts
```

### The Generation Script
```python
#!/usr/bin/env python3
import asyncio, os, re
import edge_tts

VOICE = "en-US-GuyNeural"  # Deep, warm narrator
PROJECT_DIR = os.path.dirname(os.path.abspath(__file__))
AUDIO_DIR = os.path.join(PROJECT_DIR, "audio")
os.makedirs(AUDIO_DIR, exist_ok=True)

# Clear old audio
for f in os.listdir(AUDIO_DIR):
    if f.endswith('.mp3'): os.remove(os.path.join(AUDIO_DIR, f))

with open(os.path.join(PROJECT_DIR, "index.html")) as f:
    content = f.read()

blurbs = re.findall(r"blurb:\s*'((?:[^'\\]|\\.)*)'", content)

async def generate():
    for i, blurb in enumerate(blurbs):
        text = blurb.replace("\\'", "'").replace("\\\\", "\\")
        slug = re.sub(r'[^a-z0-9]+', '-', text[:60].lower()).strip('-')
        filename = f"blurb-{i:02d}-{slug[:40]}.mp3"
        print(f"  [{i}] {filename}")
        c = edge_tts.Communicate(text, VOICE)
        await c.save(os.path.join(AUDIO_DIR, filename))

asyncio.run(generate())
```

### Voice Options
- `en-US-GuyNeural` — deep, warm, authoritative. Best for history narration. **This is what we used.**
- `en-US-ChristopherNeural` — news anchor style
- `en-US-AndrewNeural` — conversational, friendly
- `en-US-AnaNeural` — female, kid-friendly (used for Sammy SE)

### Why MP3s, Not Browser Speech
The browser's built-in `speechSynthesis` does NOT route through CarPlay. It plays through the phone speaker only. Pre-generated MP3 files played via `<audio>` element ARE media audio and DO route through CarPlay, pause podcasts/music, and work with car controls.

### AUDIO_FILES Mapping
After generating, update the mapping in index.html that connects blurb text (first 30 chars) to MP3 filenames. The generation script can output this automatically.

## Step 5: Key Features to Include

### GPS You-Are-Here Caret
- `navigator.geolocation.watchPosition()` with `enableHighAccuracy: true`
- Find nearest stop, check if next stop is closer (meaning you've passed nearest)
- Blue triangle CSS pseudo-element on `.stop.current::before`
- Everything before caret gets `.passed` class (opacity: 0.3)
- **Do NOT use localStorage/highwater marks.** They get stale when you reshuffle stops. Pure live GPS every update is more reliable.

### "How Far?" Button
- OSRM free routing API: `https://router.project-osrm.org/route/v1/driving/LNG1,LAT1;LNG2,LAT2?overview=false`
- Returns actual driving distance (meters) and duration (seconds)
- 5-second AbortController timeout — falls back to straight-line estimate
- The OSRM public server can rate-limit. Always have a fallback.

### Navigate Day from Here
- Use `saddr=Current+Location` in Apple Maps URLs so navigation starts from GPS, not the day's starting city
- Chain stops with `&daddr=Stop1&daddr=Stop2&daddr=Stop3`

### Audio Playback
- Single `new Audio()` element, reused for all blurbs
- Save `window.scrollY` before play, restore via `requestAnimationFrame` after (iOS Safari scroll reset fix)
- Button text toggles between Listen/Stop via CSS classes, NOT innerHTML (innerHTML causes reflow → scroll reset)
- Store blurbs in an `ALL_BLURBS` array, buttons reference by index: `onclick="playBlurb(3, this)"` — avoids quote escaping nightmares

### Maps
- Leaflet.js via CDN, CartoDB light tiles
- Lazy-load with IntersectionObserver (don't create 20 maps at once on mobile)
- Overview map at top with full route, day-colored segments
- Section maps with markers and dashed polylines

### Follow Button
- One-time "jump to my location" — NOT a persistent auto-scroll mode
- Auto-scroll fights the user when they try to scroll ahead to see what's coming

## Step 6: Deploy

```bash
# Initialize
git init && git add -A && git commit -m "Initial commit"

# Create repo and push
gh repo create username/trip-name --public --source=. --push

# Deploy to Vercel
npx vercel --prod --yes
```

Add to iPhone home screen: Safari → Share → Add to Home Screen.

### PWA Meta Tags
```html
<meta name="apple-mobile-web-app-capable" content="yes">
<meta name="apple-mobile-web-app-status-bar-style" content="black-translucent">
<meta name="apple-mobile-web-app-title" content="Trip Name">
<meta name="theme-color" content="#1a3a5c">
```

## Step 7: Icon

Generate with Gemini (`gemini-2.5-flash-image`). Style: two-color navy/white woodcut with diagonal stripes, matching the Carrier Pigeon aesthetic. No text. Edge-to-edge fill.

Resize with PIL: 1024 → 180 (apple-touch-icon) → 32 (favicon.ico).

## Gotchas & Hard-Won Lessons

### Data Issues
1. **Stops MUST be in geographic order.** This broke the GPS multiple times. A stop in Holbrook (west) listed after Petrified Forest (east) made the caret get stuck. Always verify order by longitude (for east-west routes) or latitude (for north-south).

2. **No 50+ mile gaps.** The nearest-stop algorithm gets stuck in long gaps. Fill them with scenic markers, food stops, or historical blurbs.

3. **Apostrophes kill everything.** `Oklahoma's` in a single-quoted JS string breaks the page. Always escape: `Oklahoma\'s`. Or better: avoid apostrophes in `desc` fields when possible.

4. **Double-check after reshuffling.** Every time you move stops between days or sections, the closing brackets (`]}`) can get mismatched. Run the JS through Node's `--check` or test in browser before pushing.

### iOS Issues
5. **iOS Safari caches aggressively** for home screen web apps. After deploying changes, users may need to delete the home screen bookmark and re-add it, or close the app completely (swipe away) and reopen.

6. **Audio playback resets scroll position on iOS.** Save `scrollY`, play, restore in `requestAnimationFrame`.

7. **`speechSynthesis` doesn't route through CarPlay.** Only `<audio>` element playback does. This is why we pre-generate MP3s.

### API Issues
8. **OSRM rate-limits.** The free server at `router.project-osrm.org` can go slow or stop responding. Always use `AbortController` with a 5-second timeout and fall back to a straight-line estimate (haversine distance × 1.25, divided by 65 mph).

9. **Don't hardcode API keys in committed files.** We exposed a Gemini key in a public repo and Google killed it within hours. Use environment variables or fetch from a credential store.

### UX Lessons
10. **The audio blurbs are the #1 feature.** Invest most of your research time here. A mediocre blurb is worse than no blurb — make each one genuinely interesting.

11. **Don't auto-scroll.** A "follow my location" mode that scrolls every 5 seconds makes the app unusable because you can't browse ahead. Use a one-tap "jump to my location" button.

12. **Every stop needs an Apple Maps link.** No exceptions. If you're mentioning a place, let the user tap to navigate there.

13. **Update the route on the fly.** The trip WILL change — different hotels, skipped stops, added detours. The app needs to be easy to update and redeploy mid-trip.

14. **Add practical stops too.** CVS with a shopping list, outlet malls with hours, breakfast spots near the hotel. The app isn't just a museum guide — it's a trip companion.

## What It Cost

| Item | Cost |
|------|------|
| Edge TTS (64 audio blurbs) | $0 |
| OSRM routing API | $0 |
| Vercel hosting | $0 |
| Leaflet + CartoDB tiles | $0 |
| Gemini icon generation | $0 (free tier) |
| **Total** | **$0** |

We initially used OpenAI TTS (~$0.45) but switched to Edge TTS with no quality loss. The Onyx voice and the Guy Neural voice are both excellent for narration.

## File Structure

```
trip-name/
├── index.html          (everything — HTML, CSS, JS, route data)
├── icon-1024.png       (app icon, full size)
├── icon-180.png        (apple-touch-icon)
├── favicon.ico         (browser tab icon)
├── audio/
│   ├── blurb-00-first-words-of-the-blurb.mp3
│   ├── blurb-01-next-blurb-text-here.mp3
│   └── ... (one MP3 per blurb, ~5-15KB each)
└── PLAYBOOK.md         (this file)
```

## Checklist for a New Trip

- [ ] Route planned: days, overnight cities, highways
- [ ] Stops researched: history, weird, food, nature, shopping
- [ ] Blurbs written for all interesting stops (2-4 sentences each)
- [ ] Data entered in `ROUTE` object, correct geographic order
- [ ] Every stop has lat/lng coordinates
- [ ] Every stop has an Apple Maps link
- [ ] No gaps over 50 miles between stops
- [ ] Audio generated with Edge TTS
- [ ] AUDIO_FILES mapping updated in index.html
- [ ] JS syntax verified (no unclosed strings, balanced brackets)
- [ ] Icon generated and sized (1024, 180, favicon)
- [ ] Deployed to Vercel
- [ ] Added to iPhone home screen
- [ ] Tested: GPS caret, audio playback, Apple Maps links, "How far?" button
