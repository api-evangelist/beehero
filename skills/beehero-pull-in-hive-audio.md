---
name: Pull BeeHero in-hive audio samples
description: Authenticate against the BeeHero API and retrieve downloadable in-hive audio recordings captured by acoustic sensors for a set of MAC addresses over a date range.
api: openapi/beehero-openapi-original.yml
base_url: https://backend.beehero.io/external
operations:
  - login
  - get_audio_sample_by_mac_address
generated: '2026-08-02'
method: generated
source: openapi/beehero-openapi-original.yml
---

# Pull BeeHero in-hive audio samples

BeeHero's acoustic sensors record inside the hive; this skill retrieves those recordings as
downloadable files.

## 1. Authenticate

Call `login` (`POST /login`) with `{ "email": ..., "password": ... }` and keep the `access_token`
from the response body. Send it as `Authorization: Bearer <access_token>` on the audio call.

## 2. Request the audio files

Call `get_audio_sample_by_mac_address` (`POST /get_audio_samples`) with the sensor MAC addresses and
the date window:

```json
{
  "mac": ["d0:cf:5e:f7:33:1d"],
  "from": "2020-10-18",
  "to": "2020-10-18"
}
```

The response is an array of `{ mac, audios[] }`, where each `audios` entry is
`{ key, url }` — `key` is the audio file key and `url` is a download link for the recording.

## 3. Download

Fetch each `url` directly. Treat the URLs as short-lived: re-request the list rather than caching
links, since no expiry is documented.

## Rules

- Audio volume is high — one sensor over one day can return many files. Keep `from`/`to` to a single
  day per call when sweeping a fleet; there is no pagination to fall back on.
- `401 Login required` — no token. `403 Invalid Token` — re-login and retry once.
- `404 Validation exception` means the request body failed validation (MAC format or date format),
  not that the hive is missing.
- Recordings are farm-operational data. Do not redistribute audio or MAC addresses outside the
  account that owns them.
- See `conventions/beehero-conventions.yml` and `errors/beehero-problem-types.yml`.
