# Home Assistant with Free My TV (LG and Samsung)

Free My TV's drawer can hold a few buttons that run things in your [Home Assistant](https://www.home-assistant.io),
such as *Movie*, *Gaming* and *Good night*. One press dims the lights, switches the TV to the right input or app, and
sets the volume. Home Assistant can also show alerts on the TV while you watch: the doorbell, the washer, anything it
knows about.

This works on the LG (webOS) and Samsung (Tizen) builds. It's optional: without it, Free My TV works as before.

You need:

- Free My TV built and installed on your TV from your own computer (README: [LG TVs](README.md#lg-tvs-webos) or
  [Samsung TVs](README.md#samsung-tvs-tizen)). The release build and the Homebrew Channel build have no Home
  Assistant buttons, because the buttons need your own address and token built in.
- A Home Assistant on the same network as the TV, that you can sign in to as an administrator.

## 1. Add your TV to Home Assistant

Home Assistant does the TV's part (inputs, apps, volume, turning off, alerts) through its own TV integration.

**LG:** Home Assistant usually finds the TV by itself. Go to **Settings → Devices & services**. If the TV shows under
*Discovered*, press **Add**. Otherwise press **Add integration**, search for **LG webOS TV** and type the TV's IP
address. The TV then asks whether to allow Home Assistant: accept it with the remote.

**Samsung:** the same, with the **Samsung Smart TV** integration.

When it's done, the TV has an entity like `media_player.living_room_tv`. Open it under **Settings → Devices & services
→ Entities** and write its entity ID down: the scripts below use it. The **Sources** it lists are the names you can
switch to (`HDMI 1`, `Netflix`, `YouTube`…).

## 2. Make a script for each button

Each button runs one script (or a scene, an automation or a button entity, if you already have them). Go to
**Settings → Automations & scenes → Scripts → Add script**, then **⋮ → Edit in YAML**, and paste one of these. Change
the entity IDs to yours (`light.living_room`, `media_player.living_room_tv`). Leave out a step you don't need.

**Movie:**

```yaml
alias: Movie mode
sequence:
  - action: light.turn_on
    target: { entity_id: light.living_room }
    data: { brightness_pct: 15 }
  - action: media_player.select_source
    target: { entity_id: media_player.living_room_tv }
    data: { source: Netflix }        # or HDMI 1, or any source the TV lists
  - action: media_player.volume_set
    target: { entity_id: media_player.living_room_tv }
    data: { volume_level: 0.2 }
```

**Gaming:**

```yaml
alias: Gaming mode
sequence:
  - action: media_player.select_source
    target: { entity_id: media_player.living_room_tv }
    data: { source: HDMI 1 }         # the console's input
  - action: light.turn_on
    target: { entity_id: light.living_room }
    data: { brightness_pct: 40 }
```

**Good night:**

```yaml
alias: Good night
sequence:
  - action: light.turn_off
    target: { entity_id: all }
  - action: media_player.turn_off
    target: { entity_id: media_player.living_room_tv }
```

Save each one. Press **Run** in Home Assistant once to check that it does what you want, before it goes on the TV.
Then open the script's **⋮ → Settings** and write down its entity ID: `script.movie_mode`, `script.gaming_mode`,
`script.good_night`.

## 3. Make a token for Free My TV

Free My TV signs in to Home Assistant with a long-lived access token.

1. In Home Assistant, click your name at the bottom left to open your profile, then the **Security** tab.
2. Under **Long-lived access tokens**, press **Create token** and name it `Free My TV`.
3. Copy the token now: Home Assistant shows it only once.

The token is a password to your home. Don't post it in a chat, an issue or a screenshot. It goes only in
`config.properties` (next step), which git leaves out. If it ever leaks, delete it on the same page and make a new
one.

## 4. Put the buttons in `config.properties`

In the Free My TV folder on your computer, open `config.properties` (the first build created it) and fill in the
three Home Assistant lines:

```properties
ha_url=http://homeassistant.local:8123
ha_token=paste-the-token-here
ha_scenes=Movie=script.movie_mode,Gaming=script.gaming_mode,Good night=script.good_night
```

- `ha_url`: the address you open Home Assistant at in a browser, without anything after the port. If
  `homeassistant.local` doesn't work from the TV, use Home Assistant's IP address (`http://<its IP>:8123`).
- `ha_token`: the token from step 3.
- `ha_scenes`: the buttons, comma separated, each as `Name=entity ID`, in the order they show. The name is what the
  button says on the TV. It also picks the symbol: a film for *movie* or *film*, a controller for *game*, a moon for
  *night* or *sleep*, a speaker for *music*, otherwise a house.

What each kind of entity does: a `script.` or `scene.` runs, an `automation.` triggers, a `button.` or
`input_button.` is pressed. Lights, switches and other entities can't be buttons directly: put them in a script.

## 5. Build and install again

**LG:**

```bash
python3 webos/build.py
python3 webos/install.py <the TV's IP>
```

**Samsung:**

```bash
python3 tizen/build.py --sign <your certificate profile>
python3 tizen/install.py <the TV's IP>
```

The build says `With 3 Home Assistant buttons.` If it says `ha_scenes needs ha_url and ha_token too`, one of those
lines is empty. A typing mistake in `ha_scenes` stops the build with a message naming the item.

The package built this way has your token inside, so don't share the `.ipk` or `.wgt` file.

## 6. Use it

Open Free My TV and press **↓** to open the drawer. The buttons are in the **Scenes** group, between the inputs and the
shortcuts. Press **OK** on one: the TV says *Home Assistant ran Movie.* when it worked.

If it says *Home Assistant didn't run …*, the end of the message tells you why:

| Message ends in | What to do |
|---|---|
| `HTTP 401` | The token is wrong or was deleted: make a new one (step 3), then build and install again. |
| `HTTP 400` or `HTTP 404` | Home Assistant has no entity with that ID: check the IDs in `ha_scenes`. |
| `ENOTFOUND`, `EHOSTUNREACH` or `timeout` | The TV can't reach `ha_url`: use Home Assistant's IP address instead of its name, and check that both are on the same network. |
| `ECONNREFUSED` | Wrong port, or Home Assistant isn't running: open `ha_url` in a browser on your computer. |
| `HTTP 0` or `no answer` (Samsung) | Home Assistant refused the TV's app because of CORS. Add `http: cors_allowed_origins` to Home Assistant's `configuration.yaml` (below), then restart Home Assistant. |

On LG, Free My TV's own small service on the TV makes the request, so CORS doesn't apply there. On Samsung, the app's
page makes it, so Home Assistant may need to allow it:

```yaml
http:
  cors_allowed_origins:
    - "*"
```

Only add this if the Samsung buttons say `HTTP 0` or `no answer`. Requests still need the token.

## Alerts on the TV while you watch

This part needs nothing in Free My TV, only the TV integration from step 1. It gives you a notify action for the TV
(on LG, a small message in the corner, over any app). Its name is under **Developer tools → Actions**: search for
`notify.` and pick the one with your TV's name, like `notify.living_room_tv`.

Three kinds of alerts work well:

- **Urgent** (the doorbell): show at once whenever the TV is on.
- **Useful** (the washer is done): wait until you're back on Free My TV's home screen, so it doesn't interrupt a film.
- **Everything else**: keep it on your phone, not the TV.

Go to **Settings → Automations & scenes → Automations → Create automation → ⋮ → Edit in YAML** and paste one of these,
with your own entity IDs:

```yaml
alias: Doorbell on the TV
triggers:
  - trigger: state
    entity_id: binary_sensor.doorbell
    to: "on"
conditions:
  - condition: state
    entity_id: media_player.living_room_tv
    state: "on"
actions:
  - action: notify.living_room_tv
    data:
      message: Someone's at the door
```

```yaml
alias: Washer done, on the TV's home screen
triggers:
  - trigger: state
    entity_id: sensor.washer_status
    to: finished
conditions:
  - condition: time          # quiet hours: nothing on the TV at night
    after: "08:00:00"
    before: "23:00:00"
actions:
  - wait_template: "{{ is_state_attr('media_player.living_room_tv', 'source', 'Free My TV') }}"
    timeout: "02:00:00"
    continue_on_timeout: false
  - action: notify.living_room_tv
    data:
      message: The washer is done
```

The second one waits up to two hours for Free My TV to be the app on screen (the TV's `source`), then shows the alert.

## Turning it off

Empty `ha_url`, `ha_token` and `ha_scenes` in `config.properties`, then build and install again: the Scenes group goes
away. Delete the token in your Home Assistant profile too.
