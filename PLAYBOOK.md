# TripTik Drive Companion — Playbook

How this app works and how to build one for any long car trip.

## What It Is

A single-page web app that turns a long drive into a guided road trip. It shows your route broken into days and sections, with stops annotated along a vertical "road" spine. Every stop has an Apple Maps link, and the interesting ones have audio blurbs you can listen to while driving — they play through CarPlay.

GPS tracks your position and shows you where you are in the list. A "How far?" button gives you real driving time/distance to any stop.

## How It Works (Technical)

### Stack
- **Single HTML file** (`index.html`) — all code, data, and styles inline
- **Leaflet.js** (CDN) — maps for each section + overview map
- **Edge TTS** — free text-to-speech from Microsoft, generates MP3 files
- **OSRM** — free routing API for driving distance/time calculations
- **Vercel** — static hosting, auto-deploys from GitHub
- No build step, no frameworks, no backend

### Data Model
All route data lives in a `ROUTE` JavaScript object:
```
ROUTE.days[].sections[].stops[] = {
  name, type, lat, lng, desc, blurb, appleMap
}
```
Stop types: `attraction`, `food`, `scenic`, `history`, `curiosity`, `truckstop`, `western`, `surplus`, `overnight`

### Key Features

**GPS You-Are-Here Caret**
- Uses `navigator.geolocation.watchPosition()` for continuous tracking
- Blue triangle caret on the left shows current position
- Everything above is grayed out (passed)
- Forward-only "highwater mark" in localStorage — never goes backward
- Algorithm: finds nearest stop by distance, only advances

**Audio Blurbs (CarPlay-Compatible)**
- Pre-generated MP3 files using Edge TTS (`en-US-GuyNeural` voice)
- Played via `<audio>` element — this is media audio, so it routes through CarPlay and pauses podcasts/music
- Browser `speechSynthesis` does NOT route through CarPlay — that's why we use MP3s
- Blurbs stored in an `ALL_BLURBS` array, buttons reference by index (avoids HTML escaping issues with quotes in text)

**Apple Maps Navigation**
- Every stop has an individual Apple Maps link
- "Navigate Day X from here" buttons use `saddr=Current+Location` so they start from GPS position, not the day's starting city
- Multi-stop links chain destinations with `&daddr=`

**"How Far?" ETA Button**
- Uses OSRM free routing API for real driving distance/time
- 5-second timeout, falls back to straight-line estimate if OSRM is slow
- Shows actual road miles and minutes, not as-the-crow-flies

**Lazy-Loaded Maps**
- IntersectionObserver creates Leaflet maps only when scrolled into view
- Prevents loading 15+ maps at once on mobile
- CartoDB light tiles for a clean, paper-map aesthetic

## How to Build One for a New Trip

### Step 1: Plan the Route
- Decide on days, overnight cities, approximate miles per day
- Use Apple Maps or Google Maps to verify driving times
- Identify the interstates/highways you'll be on

### Step 2: Research Stops
This is where the app gets its value. Search for:
- **History**: Civil War sites, Native American history, civil rights landmarks, founding-era sites
- **Weird Americana**: Roadside oddities, world's largest things, folk art, ghost towns, curiosity museums
- **Route-specific**: Route 66 landmarks, old highway culture, historic motels and gas stations
- **Food**: Vegetarian-friendly restaurants with specific addresses. Indian truck stops (dhabas) along interstates
- **Shopping**: Western wear, military surplus, outlet malls — whatever you're interested in
- **Culture**: Museums, theaters, music venues, craft traditions
- **Nature**: National parks, caverns, natural bridges, scenic overlooks

Good search queries:
- `"weird roadside attractions [highway] [state] atlas obscura"`
- `"historical stops [highway] [state] culture history"`
- `"vegetarian restaurants [city] near interstate"`
- `"Indian truck stop dhaba [highway] [state]"`

### Step 3: Write the Blurbs
Each blurb should be 2-4 sentences (~300-500 characters). Include:
- A surprising fact or vivid detail
- Historical context (when, who, why)
- Something that makes you go "huh, I didn't know that"
- Keep it conversational — these are meant to be listened to while driving

### Step 4: Build the Data
Fill in the `ROUTE` object with all stops. Critical:
- **Stops must be in geographic order** along the route or GPS tracking breaks
- Every stop needs `lat` and `lng` coordinates
- Every stop needs an `appleMap` link
- Blurbs go in the `blurb` field (use escaped single quotes)

### Step 5: Generate Audio
```bash
pip3 install edge-tts
python3 generate-audio.py
```
The script:
1. Reads all `blurb:` strings from index.html
2. Generates MP3 for each using Edge TTS (free, no API key)
3. Updates the `AUDIO_FILES` mapping in index.html

Voice options: `en-US-GuyNeural` (deep narrator), `en-US-ChristopherNeural` (news), `en-US-AndrewNeural` (conversational)

### Step 6: Deploy
```bash
git init && git add -A && git commit -m "Initial commit"
gh repo create username/trip-name --public --source=. --push
npx vercel --prod --yes
```

### Step 7: Add to iPhone Home Screen
Open the Vercel URL in Safari → Share → Add to Home Screen. The PWA meta tags give it a proper icon and full-screen experience.

## Gotchas & Lessons Learned

1. **Stop ordering matters** — Stops must be in the order you'll encounter them driving. If they're out of order, the GPS caret gets confused and shows you in the wrong place.

2. **Apostrophes in JS strings** — Blurbs with `'` need `\'` escaping. Blurbs with `"` need special handling — we use an array index system instead of inline text in onclick handlers.

3. **iOS Safari scroll reset** — Playing audio via `<audio>` can reset scroll position on iOS. Fix: save `window.scrollY` before play, restore via `requestAnimationFrame` after.

4. **Don't use persistent "follow mode"** — Auto-scrolling to GPS position every few seconds fights the user when they try to scroll ahead. Use a one-time "jump to location" button instead.

5. **OSRM rate limits** — The free OSRM server can get slow. Always have a timeout + fallback to straight-line estimate.

6. **Edge TTS is free and great** — No API key, no cost, unlimited. The `en-US-GuyNeural` voice is comparable to OpenAI's paid TTS. Use it for everything.

7. **Apple Maps `saddr=Current+Location`** — This makes navigation links start from wherever you are, not from a fixed address. Essential for mid-day use.

8. **Vercel deploys static files from git** — Audio MP3s must be committed to the repo. They're typically 5-15KB each, so even 50-60 blurbs is only ~500KB total.

## Cost

- Edge TTS: $0
- OSRM routing: $0
- Vercel hosting: $0
- Leaflet/CartoDB tiles: $0
- Total: **$0**

(We initially used OpenAI TTS at ~$0.45 total, but switched to Edge TTS with no quality loss.)
