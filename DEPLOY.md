# Deploy phone-recorder on Render

Deploy this app as one Node.js Web Service with managed HTTPS. The included `render.yaml` uses Render's Free plan and temporary storage. Phones and the dashboard may use different networks; no local CA profile, OpenSSL, or hotspot is needed. Sensor timestamps remain device timestamps. Internet latency/asymmetry can increase clock uncertainty; validate synchronization using the actual networks before an experiment.

## Free Render deployment for testing

No purchased domain or persistent disk is required. This standalone repository uses `render.yaml` at the repository root. Leave Render's Root Directory empty.

1. Publish the contents of `phone-recorder-free` to `Arhm-Iqbal/phone-recorder` (keep `package.json`, `package-lock.json`, `server.mjs`, and `render.yaml` at the repository root, with the `lib/`, `public/`, `scripts/`, and `test/` directories beside them).
2. Sign in at https://dashboard.render.com, choose **New > Blueprint**, connect GitHub, and select that repository.
3. Review the Blueprint: it must show a **Free** web service with **no disk**. Deploy it. The pairing key is generated automatically.
4. Wait for Live, then open the service's `https://...onrender.com` link. `/health` should return `{"ok":true}`. Copy `IMU_TOKEN` from the service's Environment settings and enter it once on each device. Bookmark the link.
5. Open the dashboard on the computer and the recorder in Safari on each iPhone. Select Phone 1 and Phone 2, enable sensors, and keep the pages visible and unlocked. Start when both cards show Ready, then stop, wait for Saved, and download the CSV files.

Download CSVs immediately after every recording. Free services can restart at any time and sleep after 15 minutes without inbound traffic; server files are deleted on sleep, restart, or redeployment. An active recording is interrupted by a restart. Waking the service can take about a minute. The URL remains the same. Free usage quotas apply; without a payment method, exceeding allowances can suspend service rather than incur overage charges. See https://render.com/docs/free.

## Manual Web Service settings

If you choose **New > Web Service** instead of Blueprint, Render does not automatically apply the variables in `render.yaml`. Set all of these values:

| Setting | Value |
| --- | --- |
| Repository | `Arhm-Iqbal/phone-recorder` |
| Runtime | Node |
| Root Directory | Leave empty |
| Build Command | `npm ci --omit=dev` |
| Start Command | `npm start` |
| Health Check Path | `/health` |
| Instance Type | Free |
| `NODE_VERSION` | `22` |
| `NODE_ENV` | `production` |
| `IMU_CLOUD` | `1` |
| `IMU_RECORDINGS` | `/tmp/recordings` |
| `IMU_TOKEN` | A new random secret of at least 32 characters |

Render provides `PORT`; the server binds to it on `0.0.0.0`. The same process serves the frontend and WebSockets. Do not run `npm run setup` for cloud hosting.

Generate a pairing key locally with `node -e "console.log(require('node:crypto').randomBytes(32).toString('hex'))"`. Paste it only into Render's secret setting.

## Optional paid storage

1. In `render.yaml`, change `plan: free` to `plan: starter`, set `IMU_RECORDINGS` to `/var/data/recordings`, and add the disk configuration shown below under the service.
2. Review the paid service and 1 GB disk charges before applying the updated Blueprint.
3. Once deployed, open the service's permanent HTTPS URL. Retrieve the generated `IMU_TOKEN` from the service's environment settings and enter it on the connection page once per device. Treat it as a password: it grants access to recording controls and downloads. Do not commit it or paste it in public messages.
4. Bookmark the connection page on each device. Open the dashboard on the computer and the phone recorder in Safari on each iPhone. Choose different phone numbers and enable sensors. The dashboard's private setup link can also pair a new phone.

Add this at the same indentation as `envVars`:

```yaml
    disk:
      name: recordings
      mountPath: /var/data
      sizeGB: 1
```

Deploy only source files: exclude `.local`, `.env` files, `recordings`, `node_modules`, test outputs, and any keys/configuration files. Existing local recordings are not uploaded automatically. The persistent disk stores new hosted recordings.

## Runtime settings

- `IMU_CLOUD=1`: use the hosting provider's HTTPS proxy and one HTTP/WebSocket backend port.
- `PORT`: supplied by the host; defaults to 10000.
- `IMU_TOKEN`: random secret, at least 32 characters. Cloud startup fails without it.
- `IMU_RECORDINGS`: writable recording directory. Required in cloud mode. Use `/tmp/recordings` for Free hosting or `/var/data/recordings` with the paid disk above.

Keep exactly one instance. Session coordination uses process memory. A restart or redeployment interrupts an active recording; finish and download before deploying. Files survive on the disk, but the dashboard lists only sessions from the current server process. For an older stopped session use `node scripts/export.mjs /var/data/recordings/session_UUID` in the hosting shell and retrieve its files. Interrupted sessions require manual recovery as described in README.md. Disk backups are separate from this app.

## Troubleshooting and verification

- Missing `.local/config.json` or certificate at startup: set `IMU_CLOUD=1`.
- Set IMU_TOKEN error: set a random pairing key of at least 32 characters.
- Set IMU_RECORDINGS error: set the recording path listed above.
- Cannot find package.json: keep app files at the repository root and leave Render's Root Directory empty.
- Pairing fails: use the current service's `IMU_TOKEN` on every device.
- Phone is not Ready: use Safari over HTTPS, enable motion access, select distinct phone numbers, and follow the dashboard's blocker message.

Run `npm ci --omit=dev` and `npm test` with Node.js 22 to verify locally. The cloud integration test starts without certificates, pairs two simulated phones, records through a reconnect, and downloads an authenticated CSV. This does not replace a deployed Render check or a physical iPhone trial.

References: [Render Web Services](https://render.com/docs/web-services), [Blueprint configuration](https://render.com/docs/blueprint-spec), [WebSockets](https://render.com/docs/websocket).

The default `npm start` without `IMU_CLOUD=1` retains local HTTPS/hotspot mode.
