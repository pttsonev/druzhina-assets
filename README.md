# druzhina-assets

Public static assets for the Druzhina agent fleet — currently per-agent icons.

**This repo is public on purpose.** It holds *only* harmless static images so they can be
fetched anonymously over HTTPS (e.g. Slack `icon_url`, a future dashboard, an EC2 static host).
Never put code, secrets, or anything sensitive here.

## Layout

```
agent-icons/
  vela.png      # Vela — the assistant's default Slack identity
  botev.png     # Botev
```

Icons are square PNGs with transparency (source: 1254×1254 RGBA).

## Hosting / how to use a URL

Each file is reachable at a stable raw URL:

```
https://raw.githubusercontent.com/pttsonev/druzhina-assets/main/agent-icons/<name>.png
```

Slack requires `icon_url` to return the **raw image bytes** with an `image/*` content-type over
public HTTPS — the raw URLs above satisfy that. Quick check on any host:

```bash
curl -sI '<url>' | grep -i content-type   # want: image/png
```

When the fleet moves to EC2/S3/CDN later, only the base URL changes — consumers keep the
`<name>.png` convention.
