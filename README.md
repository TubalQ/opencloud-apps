# ⚠️ Experimental - do not install

**This is experimental and not production-ready. It should not be installed on a production OpenCloud instance.**

---

# opencloud-apps

A tiny drop-in mechanism for adding apps (web extension + backend) to an
[opencloud-compose](https://github.com/opencloud-eu/opencloud-compose)
deployment. It solves the one thing that doesn't scale on its own: OpenCloud's
proxy reads a **single** `proxy.yaml`, so every backend app's routes have to live
in one file. `assemble.sh` merges each app's route fragment into that file for
you, and builds the compose file list - so adding an app is *drop a folder, run
one command, bring it up*.

## Layout

```
apps-available/<app>/      # an app you can enable
  app.yml                  # compose overlay (its services + web mount)
  routes.yaml              # this app's proxy routes (a bare list)
  web/<app>/               # the built web extension (download a release zip here)
  config/...               # any extra config (e.g. Radicale for calendar)
apps-enabled/<app>/        # apps you've turned on (copy/symlink from available)
apps.compose.yml           # mounts the generated proxy.yaml (static)
assemble.sh                # generates config/opencloud/proxy.yaml + apps-up.sh
config/opencloud/proxy.yaml  # GENERATED - do not edit
```

## Install an app

```sh
# 1. unpack this kit into your opencloud-compose checkout (next to docker-compose.yml)
# 2. enable an app (drop its folder in):
cp -r apps-available/calendar apps-enabled/
#    put the built web extension under apps-enabled/calendar/web/calendar/
#    (download the release zip and unzip it there)
# 3. assemble + bring up:
./assemble.sh
./apps-up.sh
```

To remove an app: delete its folder from `apps-enabled/`, re-run `./assemble.sh`,
bring the stack up again. To add another app, repeat step 2-3 - the routes are
merged automatically, no hand-editing of `proxy.yaml`.

## Backend images

`app.yml` defaults to the published GHCR images (`ghcr.io/.../oc-*`). To use a
locally built image instead, set e.g. `CALENDAR_BRIDGE_IMAGE` / `URL_FETCH_IMAGE`
in your environment before `./apps-up.sh`.

## Podman / SELinux hosts

Verified end-to-end on a clean OpenCloud with podman-compose. Two host notes:

- **podman**: opencloud-compose defaults `LOG_DRIVER` to `local`, which podman
  rejects. Set `LOG_DRIVER=journald` (or `json-file`) in your `.env`.
- **SELinux enforcing**: bind-mounted files (the web builds, the generated
  `proxy.yaml`, app config) need the `container_file_t` label or the container
  can't read them. Either relabel the checkout once
  (`chcon -R -t container_file_t .`) or add `:z` to the bind mounts.

## Adding your own app

Create `apps-available/<app>/` with an `app.yml` (services + a read-only mount of
your web build into `/var/lib/opencloud/web/assets/apps/<app>`) and a
`routes.yaml` (a bare YAML list of proxy route entries - no `additional_policies`
wrapper; `assemble.sh` adds it). That's the whole contract.
