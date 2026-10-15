# ZCC Log Explorer

A local web app for exploring Zscaler Client Connector logs, covering ZPA and ZIA.

Parsing happens in a local Node server that binds to `127.0.0.1` and keeps everything in memory. Logs never leave the machine.

## Run

```sh
npm install
npm start                      # http://127.0.0.1:4780
npm run start:lan              # listen on all interfaces (no auth!)
node server/index.js --load path/to/bundle.zip   # preload a bundle
```

Then either drop an exported log bundle (`.zip`, `ZSATunnel_*.log`, rotated `*.log.zip`) onto the page, or click **Load logs → Load local logs** to read the logs of the machine the server runs on.

| Env var | Default | |
|---|---|---|
| `PORT` | `4780` | |
| `HOST` | `127.0.0.1` | Set to `0.0.0.0` to expose on the network. The app has no auth. |
| `ZCC_TS_MODE` | `utc` | ZCC on macOS writes the UTC clock but labels lines with the local offset (`23:55:15(-0400)` was written at 19:55 EDT), so the clock is read as UTC. Set to `offset` to trust the label instead. |
| `ZCC_LOG_DIRS` | macOS: `/Library/Application Support/Zscaler` | Colon-separated. The Linux defaults (`/var/log/zscaler`, `/opt/zscaler/...`) are unverified guesses. |

## Views

- **Overview**: ZPA and ZIA tunnel state, ZCC version, broker, gateways, ZPA mTunnel and ZIA connection outcomes over time, and top ZPA error codes.
- **Timeline**: every log line, merged and sorted. It has search (text or regex), level, component and service (ZPA/ZIA) filters, a "hide noise" toggle for heartbeat chatter, following a ZPA `tag_id` or a ZIA connection ID, and pretty-printed JSON. It is virtualized, so it handles millions of lines.
- **ZPA Sessions**: a ZPA/ZIA state strip over time, broker sessions (`zpn_client_authenticate_ack`), lifecycle events (redirects, socket closes, segment downloads), and DNS checks sent to the broker.
- **ZPA App Segments**: every `zpn_client_app` domain the client received, with TCP/UDP ports, bypass, double encryption and ICMP settings. Also per-domain change history across downloads and incremental updates, and a "which segment would this host:port match?" tester.
- **ZPA Applications**: mTunnels grouped by destination, with success and failure counts, failure reasons, setup-time p50/p95, and the matched segment. Each connection links into the timeline.
- **ZIA Tunnel**: ZIA state (with the ZPA state on a shared time axis), Z-Tunnel 1.0/2.0 up and down events, gateway health, user notifications, gateways from the PAC, forwarding-profile transport settings (DTLS/TLS, fallback), ZIA/ZPA health-check results, and `/getTunnelBypass` lookups.
- **ZIA Web Traffic**: every connection ZSATunnel intercepted (tun adapter or local proxy), grouped by SNI/CONNECT host. Shows the path taken (Z-Tunnel 2.0, gateway CONNECT, direct), the outcome (ok, gateway HTTP error, no response), bytes up and down, and duration. ZDX probes are labeled. Each connection links to its log lines by connection ID.
- **Errors**: ZPA error codes (e.g. `BRK_MT_SETUP_FAIL_NO_POLICY_FOUND`, `APP_NOT_REACHABLE`) with troubleshooting hints, ZIA connection failures, and every ERR/WAR line grouped by pattern.

## Where the data comes from

ZPA activity is read from `ZSATunnel_*.log`. The broker protocol is logged as JSON:

```
… INF ZPN:0: Control Message Response Data: {"zpn_mtunnel_request_ack":{…}}
… DBG Writing Mtunnel request: Size: 236 Data: { "zpn_mtunnel_request": {…} }
```

`server/zpa.js` extracts these messages to build sessions, segments and mTunnels.

ZIA connections are logged per connection ID (`ID=123, TUN-Proxy: connection to …`, `SNI-Host=…`, `PAC Parse Action: …`, `~ZTCPServerConnection state=closed … srv_bytes=…`), and tunnel state changes appear as `Changing ZIA state from: X to Y`. `server/zia.js` assembles these into connections and lifecycle events.

The UI shows every time in the viewer's browser time zone; the timeline detail pane also shows the timestamp exactly as written in the log.

ZCC version and user notifications come from `ZSAService_*.log`. Segment matching in the UI approximates the client's most-specific-match rule; the broker's access policy still makes the final decision.

## Development

```sh
npm run dev   # restart on change
npm test      # parser/analyzer tests (synthetic fixtures)
```

`samples/` is gitignored. Put real logs there for local testing, but never commit them: they contain usernames, IPs, and internal hostnames.
