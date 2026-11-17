# UniFi Protect camera wall — private LAN HTTPS

This app shows four UniFi Protect feeds through one private HTTPS address. It is designed for a trusted home network and does not use Cloudflare, public DNS, port forwarding, or internet ingress.

## How it is arranged

```text
Protect NVR --private RTSP--> MediaMTX --private Docker HLS--> Caddy HTTPS
                                                                    |
                                      desktop/iPad <--home LAN------+
```

Only TCP port `8443` is published by Docker. The RTSP URLs, MediaMTX port, and stream tokens are not exposed to browsers or other LAN clients.

## HTTPS in plain language

HTTPS encrypts traffic and lets the browser verify that it reached the expected server. For a private IP such as `192.168.0.203`, a normal public certificate provider cannot perform that verification. Caddy therefore creates a private certificate authority specifically for this app.

The app is encrypted immediately, but each client must trust the Caddy root certificate once. Until that is done, the browser will display a certificate warning. Bypassing the warning is not enough for reliable embedded video; install and trust the root certificate.

## Configure the app

1. Keep the four converted Protect URLs in `.env`:

   ```dotenv
   CAMERA_1_URL=rtsp://NVR_LAN_IP:7447/CAMERA_1_TOKEN
   CAMERA_2_URL=rtsp://NVR_LAN_IP:7447/CAMERA_2_TOKEN
   CAMERA_3_URL=rtsp://NVR_LAN_IP:7447/CAMERA_3_TOKEN
   CAMERA_4_URL=rtsp://NVR_LAN_IP:7447/CAMERA_4_TOKEN
   ```

   Protect's copied URL normally changes from:

   ```text
   rtsps://NVR_LAN_IP:7441/TOKEN?enableSrtp
   ```

   to this trusted-LAN MediaMTX source:

   ```text
   rtsp://NVR_LAN_IP:7447/TOKEN
   ```

2. Add the address clients will use. This desktop currently uses `192.168.0.203`:

   ```dotenv
   CAMERA_APP_HOST=192.168.0.203
   CAMERA_HTTPS_PORT=8443
   ```

   Use a stable/reserved LAN IP. The value must exactly match the address in the browser. When deploying to Ubuntu, replace it with the server's stable LAN IP before starting the stack.

## Start or recreate it

From this folder:

```powershell
docker compose down
docker compose up -d --force-recreate
docker compose logs --tail=100 gateway media
```

Open the dashboard on this desktop:

```text
https://localhost:8443/
```

The desktop-local address has its own valid local certificate and does not
depend on the Windows LAN firewall. Use the IP-based address from the iPad or
another device.

Individual camera player URLs are:

```text
https://192.168.0.203:8443/cam1/
https://192.168.0.203:8443/cam2/
https://192.168.0.203:8443/cam3/
https://192.168.0.203:8443/cam4/
```

The first request can take several seconds because each stream starts on demand and HLS buffers its first segments.

## Trust the certificate on Windows

After the stack starts, export Caddy's public root certificate:

```powershell
docker compose cp gateway:/data/caddy/pki/authorities/local/root.crt ./caddy-local-root.crt
```

Open an Administrator PowerShell in this folder and install it:

```powershell
certutil -addstore -f ROOT .\caddy-local-root.crt
```

Close and reopen the browser. The app URL should now show a normal secure connection without a warning.

Only the public root certificate is exported. The private CA key remains protected in Docker's `caddy_data` volume. Do not export or share anything named `root.key`.

## Trust the certificate on the wall iPad

1. Export `caddy-local-root.crt` with the command above.
2. Transfer that `.crt` file to the iPad using AirDrop, Files, or another private method and open it.
3. In Settings, install the downloaded profile.
4. Go to **Settings → General → About → Certificate Trust Settings**.
5. Under **Enable Full Trust for Root Certificates**, enable **Caddy Local Authority**.
6. Open Safari and visit `https://192.168.0.203:8443/`.

Installing the profile alone is not enough on iPadOS; full trust must also be enabled.

## Windows firewall

Other devices need inbound TCP `8443` on the Windows **Private** network profile. If it is not already allowed, run once from Administrator PowerShell:

```powershell
New-NetFirewallRule -DisplayName "Protect Camera Wall HTTPS" -Direction Inbound -Action Allow -Protocol TCP -LocalPort 8443 -Profile Private
```

Do not create a router port-forward for this port.

## Move to the Ubuntu server

1. Copy this project folder except `.env`, `caddy-local-root.crt`, and Docker data.
2. Create a new server `.env` containing the four RTSP URLs, the Ubuntu server's stable LAN IP, and port `8443`.
3. Allow port `8443` only from the home subnet. For a `192.168.0.0/24` home network:

   ```bash
   sudo ufw allow from 192.168.0.0/24 to any port 8443 proto tcp
   ```

4. Start the stack with `docker compose up -d`.
5. Export the Ubuntu deployment's new `root.crt` and install it on the iPad. A new Caddy data volume creates a new local certificate authority, so the desktop test certificate will not automatically trust the Ubuntu deployment.

If you intentionally migrate the `caddy_data` volume to Ubuntu, the same root authority can be retained. Treat that volume as sensitive because it contains the CA private key.

## Embedding or linking from another local app

Use the HTTPS camera URLs above. If the other app itself is HTTPS, these HTTPS feeds avoid the browser's mixed-content block. The client still has to trust the same Caddy root certificate.

The camera player is on a different origin when the host or port differs from the other app. A normal link works without extra changes. An iframe can also require that app's Content Security Policy permit this origin in `frame-src`; that is a browser policy in the embedding app, not an HTTPS problem in this camera service.

## Troubleshooting

- **Certificate warning:** Confirm `CAMERA_APP_HOST` exactly matches the host/IP in the URL, then install and fully trust the exported root certificate.
- **Desktop works but iPad cannot connect:** Allow inbound TCP `8443` in the host firewall and verify both devices are on the same non-guest LAN/VLAN.
- **Dashboard loads but a camera is blank:** Open its `/cam1/` URL directly and inspect `docker compose logs media`. Confirm its RTSP URL uses `rtsp`, port `7447`, and no `?enableSrtp` suffix.
- **`source ... timed out`:** MediaMTX cannot reach or authenticate to the Protect NVR. This is upstream of HTTPS.
- **Video starts slowly:** On-demand HLS needs initial stream startup and buffering; several seconds is normal.
- **The IP changed:** Reserve the server address in DHCP, update `CAMERA_APP_HOST`, recreate the gateway, and use the matching URL.
- **Endar or another HTTPS page refuses an iframe:** Add `https://SERVER_IP:8443` to that app's `frame-src` policy. Do not disable browser security globally.

## Security boundary

There is no application login. Anyone on a network that can reach TCP `8443` can view the feeds. Keep the port limited to the private home LAN, do not port-forward it, and do not add public ingress. HTTPS encrypts and authenticates the connection; it does not add user authentication.
