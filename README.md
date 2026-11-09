# arkime-sourceip

[SourceIP.io](https://sourceip.io/) enrichment for [Arkime](https://arkime.com/). It tells you who operates a public IP and
how they use it (`self_operated`, `shared_platform` or `general_use`), which is not the same question GeoIP or ASN data answers.

| Component | File | What it adds |
|---|---|---|
| Cont3xt integration | `cont3xt/integrations/sourceip/index.js` | A SourceIP.io card: providers, ASN origins with RPKI status, registry allocation |
| WISE source | `wise/source.sourceip.js` | A `sourceip.provider` session field, filled at capture time |

The two parts are independent, so you can install either one. Tested with Arkime 6.8.0.

## Privacy

With the WISE source enabled, **every public IP your capture sees is sent to sourceip.io** (results are cached for a day).
Private, loopback, link-local, CGNAT, multicast and documentation ranges are never sent. The Cont3xt integration only sends the
IPs you search for. Check that this is acceptable for your network before enabling the WISE source.

## API key

The API works without a key at 100 lookups a day per client IP. A free key from <https://sourceip.io/api-key> raises that to
1,000 a day. Keyed routes are limited to 5 requests per second (burst 10).

Keyed lookups need **both** values. If either is missing, the code silently uses the anonymous endpoint.

    SOURCEIP_API_KEY=your-key
    SOURCEIP_ENDPOINT=https://api.sourceip.io

Only the WISE source batches lookups, and only with a key. Without one it makes one request per IP.

The Cont3xt integration and the WISE source share the key's daily quota.

## Install with Docker Compose

Clone this repo next to your Compose file, then add the mounts and variables. The service names `wise` and `cont3xt` are
examples; use your own.

```yaml
services:
  wise:
    environment:
      SOURCEIP_API_KEY: ${SOURCEIP_API_KEY:-}
      SOURCEIP_ENDPOINT: ${SOURCEIP_ENDPOINT:-}
    volumes:
      - ./arkime-sourceip/wise/source.sourceip.js:/opt/arkime/wiseService/source.sourceip.js:ro

  cont3xt:
    environment:
      SOURCEIP_API_KEY: ${SOURCEIP_API_KEY:-}
      SOURCEIP_ENDPOINT: ${SOURCEIP_ENDPOINT:-}
    volumes:
      - ./arkime-sourceip/cont3xt/integrations/sourceip:/opt/arkime/cont3xt/integrations/sourceip:ro
```

No image rebuild is needed. Arkime loads these files at startup: WISE globs `source.*.js`, and Cont3xt globs
`integrations/**/index.js`.

## Install without Docker

The files use relative `require` paths, so they must be in these exact locations:

    cp wise/source.sourceip.js /opt/arkime/wiseService/
    cp -r cont3xt/integrations/sourceip /opt/arkime/cont3xt/integrations/

Set the two environment variables for those services, or set the key in configuration as described below.
Reinstalling or upgrading Arkime may replace these directories, so copy the files again afterwards.

## Configure the WISE source

Add to `wiseService.ini`:

```ini
[sourceip]
enabled=true
```

| Option | Default | Meaning |
|---|---|---|
| `enabled` | `false` | Must be `true` for the source to load |
| `key` | `SOURCEIP_API_KEY` | API key |
| `endpoint` | `SOURCEIP_ENDPOINT` | Keyed API endpoint |
| `field` | `sourceip.provider` | Arkime field to fill |
| `dailyBudget` | `800` keyed, `60` anonymous | Addresses this source may look up per UTC day. The counter is in memory and resets when WISE restarts |
| `reserve` | `100` keyed, `0` anonymous | Pause when the server reports this many lookups or fewer remaining, leaving some for Cont3xt |
| `batchWaitMs` | `500` | How long to collect IPs into one batch |
| `timeoutMs` | `4000` | Hard deadline for each API call |
| `cacheAgeMin` | `1440` | WISE result cache age |
| `trace` | `false` | Log every public-IP lookup and its outcome |

Keep `batchWaitMs + timeoutMs` under 5 seconds, which is WISE's own response timeout. A call that fails gives its IPs no
result, which is not cached by WISE, so they are retried later.

Then enable WISE in Arkime's `config.ini`:

```ini
[default]
plugins=wise.so
viewerPlugins=wise.js
wiseURL=http://127.0.0.1:8081
```

Restart WISE first, wait until it answers, and then restart capture:

    curl -s http://127.0.0.1:8081/sources

**Start order matters.** If capture starts while WISE is down, it keeps running but no session gets the field until capture is
restarted. Look for `Couldn't connect to WISE` in the capture log. This can also happen after a host reboot.

In the viewer, add the **SourceIP.io Provider** column from the Sessions column picker. Only new sessions get the field. To find
sessions that have it:

    sourceip.provider == EXISTS!

## Configure the Cont3xt integration

Restart Cont3xt after installing. The integration is then listed as **SourceIP.io**. Cont3xt only queries the integrations you
select, so tick it in the Integrations panel or add it to a saved view.

Each user can also set their own key and endpoint under Settings > Integrations. These take priority over the environment
variables. Leave them unset, rather than saved as empty, to use the environment values.

Results are cached for a day. A 400 or 404 shows as no result. A 429 or other error is logged and also shows no data, so check
the Cont3xt log if a card is empty.

### Icon

The repo does not include a logo, because it belongs to SourceIP.io. The card works without one. To add an icon, place a file
named `icon.png` in the `sourceip` integration directory. It must be `.png`, `.jpg` or `.jpeg`, because Cont3xt only serves those
types from `integrations/`.

## Optional: right-click link

To open an IP on sourceip.io from the viewer's menu, add this to `wiseService.ini`. It is a plain link and uses no quota.

```ini
[right-click]
SourceIP=url:https://sourceip.io/lookup?ip=%TEXT%;name:SourceIP.io;category:ip
```

## Verify

    # WISE: the source loaded and answers for an IP
    docker compose logs wise | grep sourceip
    curl -s http://127.0.0.1:8081/sourceip/ip/8.8.8.8

    # Cont3xt: the integration is registered and returns data
    curl -sk --digest -u admin:PASSWORD https://localhost:3218/api/integration | grep -c SourceIP.io
    curl -sk --digest -u admin:PASSWORD -H 'content-type: application/json' \
      -d '{"query":"8.8.8.8","doIntegrations":["SourceIP.io"]}' https://localhost:3218/api/integration/search

The WISE startup line shows which mode is active: `keyed batch https://api.sourceip.io/v1/lookup/batch` or `anonymous`.

## Troubleshooting

| Symptom | Likely cause |
|---|---|
| WISE logs `anonymous` although a key is set | `SOURCEIP_ENDPOINT` is missing |
| `sourceip batch timeout after 4000 ms` | The API did not respond in time. Those IPs get no result and are retried later |
| `sourceip no result: daily budget exhausted` | `dailyBudget` reached; resets at UTC midnight |
| `only N lookups left, pausing until ...` | The server's remaining count reached `reserve` |
| No sessions have the field | WISE was down when capture started, or `plugins=wise.so` is missing |
| Long-lived sessions lack the field | Mid-save path; see [Improving coverage](#improving-coverage) |
| Cont3xt shows "No integrations selected" | Tick SourceIP.io or choose a view |

## API routes used

| Route | Used by |
|---|---|
| `GET https://sourceip.io/api/v1/lookup?ip=` | Both, without a key |
| `GET https://api.sourceip.io/v1/lookup?ip=` | Cont3xt, with a key |
| `POST https://api.sourceip.io/v1/lookup/batch` | WISE, with a key |

The `/v1/asn/*` routes are not used. See the [SourceIP.io API docs](https://sourceip.io/api).

## License

Apache-2.0. See [LICENSE](LICENSE).
