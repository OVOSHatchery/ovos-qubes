# ovos-qubes

setting up a hardened ovos-core under [QubesOS](https://www.qubes-os.org)

![Logo](ovos-qubes.png)

* [Architecture](#architecture)
* [Creating OVOS Qubes](#creating-ovos-qubes)
    + [template-ovos-base](#template-ovos-base)
    + [ovos-backend](#ovos-backend)
    + [ovos-bus](#ovos-bus)
    + [ovos-audio](#ovos-audio)
    + [ovos-skills](#ovos-skills)
    + [ovos-speech](#ovos-speech)
    + [ovos-phal](#ovos-phal)
    + [ovos-gui](#ovos-gui)
    + [ovos-gui-client](#ovos-gui-client)
* [Connecting the Qubes](#connecting-the-qubes)
* [Hardening](#hardening)
    + [Firewall ovos-backend](#firewall-ovos-backend)
    + [Choose an offline STT](#choose-an-offline-stt)
* [How to](#how-to)
    + [Enable ovos-backend](#enable-ovos-backend)
    + [Expose ovos-bus to other qubes](#expose-ovos-bus-to-other-qubes)
    + [Expose ovos-backend to other qubes](#expose-ovos-backend-to-other-qubes)
    + [Expose ovos-gui to other qubes](#expose-ovos-gui-to-other-qubes)
* [Troubleshooting](#troubleshooting)
    + [No audio in ubuntu template](#no-audio-in-ubuntu-template)

## Architecture

The aim here is to bring maximum isolation between services and reduce the chance that a misbehaved component can mess
with the system

Considerations:

- one ovos service corresponds to one qube
- whenever possible a service is disconnected from the internet
- whenever possible a service is firewalled to only allow specific outgoing connections

![connections](connected_qubes.png)

    gray - no internet
    green - only whitelisted outgoing connections
    yellow - internet access

- sys-ovos-firewall
    - DispVM
    - set firewall rules
    - can be connected [sys-vpn](https://github.com/Qubes-Community/Contents/blob/master/docs/configuration/vpn.md)
    - can use [mirage firewall unikernel](https://github.com/mirage/qubes-mirage-firewall)
    - all networked ovos qubes connect here
- template-ovos-base
    - templateVM
    - /etc/mycroft/mycroft.conf lives here
    - system packages installed/updated here
- ovos-backend
    - AppVM
    - ovos-local-backend installed as user
    - sys-ovos-firewall needed for API services / remote STT
    - 6712 port exposed to other ovos qubes
    - can be configured as a [selene proxy](https://github.com/OpenVoiceOS/OVOS-local-backend#selene-proxy) if desired
- ovos-bus
    - AppVM
    - ovos-bus installed as user
    - sys-ovos-firewall not needed
    - 8181 port exposed to other ovos qubes
- ovos-gui
    - AppVM
    - ovos-gui installed as user
    - sys-ovos-firewall not needed
    - 18181 port exposed to ovos-gui-client qubes
- ovos-audio
    - AppVM
    - ovos-audio installed as user
    - sys-ovos-firewall needed for stream playback / remote TTS
- ovos-speech
    - AppVM
    - ovos-speech installed as user
    - sys-ovos-firewall needed for remote STT
    - mic needs to be explicitly attached
- ovos-skills
    - AppVM
    - ovos-skills installed as user
    - sys-ovos-firewall needed for internet skills
- ovos-phal
    - AppVM
    - ovos-PHAL installed as user
    - dedicated plugins for qubes need to be written ⚠️
- ovos-gui-client
    - StandaloneVM
    - based on [ubuntu template](https://qubes.3isec.org/Templates_4.1)
    - mycroft-gui installed as user from source
    - sys-ovos-firewall needed for stream playback / web browsing / remote pictures

## Creating OVOS Qubes

![qubes manager](ovos-qubes-manager.png)

### template-ovos-base

- clone a debian template VM as `template-ovos-base`
- debloat the image
  ```bash
  sudo apt purge firefox-esr thunderbird keepassxc nautilus
  sudo apt purge emacs*
  sudo apt purge vim*
  sudo apt autoremove
  ```
- install system packages
  ```bash
  sudo apt-get install python3-pip swig libfann-dev portaudio19-dev libpulse-dev libespeak-ng1
  sudo apt-get install qubes-core-agent-dom0-updates qubes-core-agent-passwordless-root
  ```
- create mycroft.conf
  ```bash
  sudo mkdir /etc/mycroft
  sudo nano /etc/mycroft/mycroft.conf
  ```
- configure to your liking
  ```json
  {
    "lang": "en-us",
    "time_format": "full",
    "date_format": "DMY",
    "gui": {
      "extension": "smartspeaker",
      "idle_display_skill": "skill-ovos-homescreen.openvoiceos"
    },
    "stt": {"module": "ovos-stt-plugin-vosk"},
    "tts": {"module": "ovos-tts-plugin-mimic3"},
    "padatious": {"regex_only": true},
    "server": {
      "disabled": true
    },
    "debug": true,
    "log_level": "DEBUG"
  }
  ```

### ovos-backend

- create `ovos-backend` qubes from `template-ovos-base`
- select `sys-ovos-firewall` as NetVM
- (optional) make this qube launch on boot
- install ovos-local-backend (no sudo!)
  ```bash
  pip install git+https://github.com/OpenVoiceOS/OVOS-local-backend
  ```
- install extra STT plugins
    - system dependencies (if any) need to be installed in `template-ovos-base`
    - install recommended plugins
  ```bash
  pip install ovos-stt-plugin-server 
  pip install ovos-stt-plugin-vosk
  ```
- configure backend `nano ~/.config/json_database/ovos_backend.json`
  ```
  {
    "stt": {
      "module": "ovos-stt-plugin-server",
      "ovos-stt-plugin-server": {"url": "https://stt.openvoiceos.com/stt"}
    },
    "selene": {
      "enabled": false,
      "proxy_pairing": true
    },
    "backend_port": 6712,
    "skip_auth": true,
    "geolocate": true,
    "override_location": false,
    "api_version": "v1",
    "data_path": "~",
    "record_utterances": false,
    "record_wakewords": false,
    "wolfram_key": "$KEY",
    "owm_key": "$KEY",
    "lang": "en-us",
    "date_format": "DMY",
    "system_unit": "metric",
    "time_format": "full",
    "default_location": {
      "city": {"...": "..."},
      "coordinate": {"...": "..."},
      "timezone": {"...": "..."}
    }
  }  
  ```

  ⚠️ only ovos-skills will have a `identity2.json`, `"skip_auth"` needs to be set, pairing is not shared across ovos
  qubes ⚠️


- create auto_start.sh `nano auto-start.sh`
  ```bash
  /home/user/.local/bin/ovos-local-backend >> /home/user/backend.log 2>&1 &
  ```
- create .desktop file to autostart ovos-backend when the VM
  boots `nano /home/user/.config/autostart/ovos-backend.desktop`
  ```
  [Desktop Entry]
  Name=OVOS Local Backend
  Exec=bash /home/user/auto-start.sh
  Type=Application
  StartupNotify=true
  NoDisplay=true
  ```
- [Enable ovos-backend](#enable-ovos-backend) in `template-ovos-base` or in every independent `ovos-XXX` qube
- (optional) [setup firewall rules](#firewall-ovos-backend), only allow outgoing connections to the domains of provided
  services

### ovos-bus

- create `ovos-bus` qubes from `template-ovos-base`
- set none as NetVM, this service does not need to reach to the internet
- (optional) make this qube launch on boot
- install ovos-bus (no sudo!)
  ```bash
  pip install ovos-core[bus]
  ```
- create auto_start.sh `nano auto-start.sh`
  ```bash
  /home/user/.local/bin/mycroft-messagebus >> /home/user/bus.log 2>&1 &
  ```
- create .desktop file to autostart ovos-bus when the VM
  boots `nano /home/user/.config/autostart/mycroft-messagebus.desktop`
  ```
  [Desktop Entry]
  Name=OVOS Messagebus
  Exec=bash /home/user/auto-start.sh
  Type=Application
  StartupNotify=true
  NoDisplay=true
  ```
- (optional) [disable ovos-backend](#enable-ovos-backend)

### ovos-audio

- create `ovos-audio` qubes from `template-ovos-base`
- select `sys-ovos-firewall` as NetVM
- (optional) make this qube launch on boot
- install ovos-audio (no sudo!)
  ```bash
  pip install ovos-core[audio,tts,audio_backend]
  ```
- install extra TTS plugins
    - system dependencies (if any) need to be installed in `template-ovos-base`
    - install recommended plugins
  ```bash
  pip install ovos-tts-plugin-marytts
  pip install ovos-tts-plugin-mimic3
  ```
- create auto_start.sh `nano auto-start.sh`
  ```bash
  /home/user/.local/bin/mycroft-audio >> /home/user/audio.log 2>&1 &
  ```
- create .desktop file to autostart ovos-audio when the VM
  boots `nano /home/user/.config/autostart/mycroft-audio.desktop`
  ```
  [Desktop Entry]
  Name=OVOS Audio Service
  Exec=bash /home/user/auto-start.sh
  Type=Application
  StartupNotify=true
  NoDisplay=true
  ```
- [expose ovos-bus to ovos-audio](#expose-ovos-bus-to-other-qubes)
- (optional) [Enable ovos-backend](#enable-ovos-backend)
    - you need to copy identity2.json from ovos-skills to keep the device uuid
- (optional) setup firewall rules, only allow outgoing connections to the domains of remote TTS

### ovos-skills

- create `ovos-skills` qubes from `template-ovos-base`
- select `sys-ovos-firewall` as NetVM
- (optional) make this qube launch on boot
- install ovos-skills (no sudo!)
  ```bash
  pip install ovos-core[skills]
  ```
- install skills
    - system dependencies (if any) need to be installed in `template-ovos-base`
    - install recommended skills (TODO add list)
- create auto_start.sh `nano auto-start.sh`
  ```bash
  /home/user/.local/bin/mycroft-skills >> /home/user/skills.log 2>&1 &
  ```
- create .desktop file to autostart ovos-skills when the VM
  boots `nano /home/user/.config/autostart/mycroft-skills.desktop`
  ```
  [Desktop Entry]
  Name=OVOS Skills Service
  Exec=bash /home/user/auto-start.sh
  Type=Application
  StartupNotify=true
  NoDisplay=true
  ```
- [expose ovos-bus to ovos-skills](#expose-ovos-bus-to-other-qubes)
- (optional) [Enable ovos-backend](#enable-ovos-backend)

### ovos-speech

- create `ovos-speech` qubes from `template-ovos-base`
- (optional) select `sys-ovos-firewall` as NetVM
    - not needed if you [choose an offline STT](#choose-an-offline-stt)
- (optional) make this qube launch on boot
- install ovos-speech (no sudo!)
  ```bash
  pip install ovos-core[stt]
  ```
- install extra STT plugins
    - system dependencies (if any) need to be installed in `template-ovos-base`
    - install recommended plugins
  ```bash
  pip install ovos-stt-plugin-selene 
  pip install ovos-stt-plugin-vosk
  pip install ovos-stt-plugin-chromium
  ```
- create auto_start.sh `nano auto-start.sh`
  ```bash
  /home/user/.local/bin/mycroft-speech-client >> /home/user/stt.log 2>&1 &
  ```
- create .desktop file to autostart ovos-speech when the VM
  boots `nano /home/user/.config/autostart/mycroft-stt.desktop`
  ```
  [Desktop Entry]
  Name=OVOS Speech Client
  Exec=bash /home/user/auto-start.sh
  Type=Application
  StartupNotify=true
  NoDisplay=true
  ```
- [expose ovos-bus to ovos-speech](#expose-ovos-bus-to-other-qubes)
- (optional) [Enable ovos-backend](#enable-ovos-backend)
    - integrates with selene stt plugin
        - if using selene stt plugin you can set NetVM to none
    - you need to copy identity2.json from ovos-skills to keep the device uuid
- [attach microphone](https://www.qubes-os.org/doc/how-to-use-devices/#attaching-devices)

### ovos-phal

- create `ovos-phal` qubes from `template-ovos-base`
- select `sys-ovos-firewall` as NetVM
    - depending on plugins installed this may be set to None
- (optional) make this qube launch on boot
- install ovos-phal (no sudo!)
  ```bash
  pip install ovos-phal
  ```
- install phal plugins
    - system dependencies (if any) need to be installed in `template-ovos-base`
    - install recommended plugins (TODO add list)
- create auto_start.sh `nano auto-start.sh`
  ```bash
  /home/user/.local/bin/ovos_PHAL >> /home/user/phal.log 2>&1 &
  ```
- create .desktop file to autostart ovos-phal when the VM boots `nano /home/user/.config/autostart/PHAL.desktop`
  ```
  [Desktop Entry]
  Name=OVOS PHAL Service
  Exec=bash /home/user/auto-start.sh
  Type=Application
  StartupNotify=true
  NoDisplay=true
  ```
- [expose ovos-bus to ovos-phal]((#expose-ovos-bus-to-other-qubes))
- (optional) [disable ovos-backend](#enable-ovos-backend)

### ovos-gui

- create `ovos-gui` qubes from `template-ovos-base`
- set none as NetVM, this service does not need to reach to the internet
- (optional) make this qube launch on boot
- install ovos-gui (no sudo!)
  ```bash
  pip install ovos-core[gui]
  ```
- create auto_start.sh `nano auto-start.sh`
  ```bash
  /home/user/.local/bin/mycroft-gui-service >> /home/user/gui.log 2>&1 &
  ```
- create .desktop file to autostart ovos-gui when the VM
  boots `nano /home/user/.config/autostart/mycroft-gui-messagebus.desktop`
  ```
  [Desktop Entry]
  Name=OVOS GUI Messagebus
  Exec=bash /home/user/auto-start.sh
  Type=Application
  StartupNotify=true
  NoDisplay=true
  ```
- [expose ovos-bus to ovos-gui]((#expose-ovos-bus-to-other-qubes))
- (optional) [disable ovos-backend](#enable-ovos-backend)

### ovos-gui-client

- install [community](https://www.qubes-os.org/doc/templates/#community) ubuntu template in dom0
    - download [here](https://qubes.3isec.org/Templates_4.1) to some qube, eg `dl_qube`
    - copy downloaded file to
      dom0 `qvm-run --pass-io dl_qube 'cat /home/user/Downloads/focal.rpm' > /home/user/focal.rpm`
    - install `sudo dnf install focal.rpm`
    - ⚠️ [Fix no audio bug](#no-audio-in-ubuntu-template)
- create `ovos-gui-client` qubes from `focal` as a StandaloneVM
- select `sys-ovos-firewall` as NetVM
- install mycroft-gui
  ```bash
  git clone https://github.com/MycroftAI/mycroft-gui
  cd mycroft-gui
  ./dev_setup.sh
  ```
- [expose ovos-bus to ovos-gui-client]((#expose-ovos-bus-to-other-qubes))
- [expose ovos-gui to ovos-gui-client]((#expose-ovos-bus-to-other-qubes))
- launch `mycroft-gui-app` from this qube when wanted
    - (optional) launch on qube on boot
    - (optional) launch mycroft gui on VM startup

## Connecting the Qubes

We need to [open a TCP port to other network-isolated qubes](https://www.qubes-os.org/doc/firewall/#opening-a-single-tcp-port-to-other-network-isolated-qube)
for ovos-bus, ovos-gui and ovos-backend

![exposed ports](ovos-policy.png)

- [expose port 8181]((#expose-ovos-bus-to-other-qubes))
    - from `ovos-bus` to `ovos-audio`
    - from `ovos-bus` to `ovos-speech`
    - from `ovos-bus` to `ovos-skills`
    - from `ovos-bus` to `ovos-gui`
    - from `ovos-bus` to `ovos-gui-client`
    - from `ovos-bus` to `ovos-phal`
- [expose port 6712](#expose-ovos-backend-to-other-qubes)
    - from `ovos-backend` to `ovos-skills`
    - (optional) from `ovos-backend` to `ovos-speech`
        - required if using selene plugin
- [expose port 18181](#expose-ovos-gui-to-other-qubes)
    - from `ovos-gui` to `ovos-gui-client`

## Hardening

### Firewall ovos-backend

ovos-backend can be restricted to only allow certain outgoing connections, this is a very good idea since we know
exactly which services this qube needs to connect and why

![backend firewall](ovos-backend-fw.png)

| service        | domain                      | used for                  | required by                   | disabled by                                                        |
|----------------|-----------------------------|---------------------------|-------------------------------|--------------------------------------------------------------------|
| IPify          | api.ipify.org               | ip geolocation            | `"geolocate": true`           | `"override_location": true`                                        |                               |
| OpenWeatherMap | api.openweathermap.org      | weather api               | `"owm_key": "your_key"`       | `"proxy_weather": true`                                            |
| OpenStreetMap  | nominatim.openstreetmap.org | geolocation api           | default                       | `"proxy_geolocation": true`                                        |
| WolframAlpha   | api.wolframalpha.com        | wolfram alpha api         | `"wolfram_key": "your_key"`   | `"proxy_wolfram": true`                                            |
| OpenVoiceOS    | openvoiceos.com             | free microservice proxies | default                       | `"xxx_key": "your_key"` or `"proxy_xxx": true` (wolfram + weather) |
| Email Service  | ?                           | SMTP email server         | `"email": {"..."}`            | `"proxy_email": true`                                              |
| STT Service    | ?                           | remote STT transcription  | some stt plugins              | default                                                            |
| Selene         | api.mycroft.ai              | selene proxy integration  | `"selene": {"enabled": true}` | default                                                            |

### Choose an offline STT

There are some offline STT options listed below, if any of those is acceptable to you then you should configure
ovos-qubes to use it, this way your speech will never leave your computer

Option 1 - directly in ovos-speech

- configure `~/.config/mycroft/mycroft.conf` to use your offline-stt-plugin
- ensure ovos-speech has no NetVM
- (optional) disable ovos-backend
    - ovos-backend only needed if you want metrics upload

Option 2 - in ovos-backend

- configure `~/.config/mycroft/mycroft.conf` to use your ovos-backend
- configure `~/.config/mycroft/mycroft.conf` to use your selene-plugin
- configure `~/.config/json_database/ovos_backend.json` to use your offline-stt-plugin
- [expose ovos-backend to ovos-speech](#expose-ovos-backend-to-other-qubes)
- ensure ovos-speech has no NetVM
    - ovos-backend will still be available

## How to

### Enable ovos-backend

By default ovos-core does not use a backend, we need to explicitly enable ovos-backend in mycroft.conf

to enable ovos-backend edit mycroft.conf in the desired qubes with the following

```json
  {
  "server": {
    "disabled": false,
    "url": "http://0.0.0.0:6712",
    "version": "v1",
    "update": true,
    "metrics": true
  },
  "listener": {
    "wake_word_upload": {
      "url": "http://0.0.0.0:6712/precise/upload"
    }
  },
  "opt_in": true
}
```

to disable backend edit mycroft.conf with the following

```json
  {
  "server": {
    "disabled": true
  }
}
```

Option 1 - globally, in template-ovos-base

- edit `/etc/mycroft/mycroft.conf`
- [expose ovos-backend to every ovos-XXX qubes](#expose-ovos-backend-to-other-qubes)

Option 2 - selectively, per qube

- edit `~/.config/mycroft/mycroft.conf` in ovos-XXX
- enable backend in ovos-skills
    - [expose ovos-backend to ovos-skills](#expose-ovos-backend-to-other-qubes)
    - needed for pairing process
    - needed for metrics
    - needed for device configuration via backend (location, preferences, skill settings)
    - needed for skills that use the microservices (email, geolocation, weather, wolfram alpha)
- (optional) enable backend in ovos-audio
    - [expose ovos-backend to ovos-audio](#expose-ovos-backend-to-other-qubes)
    - needed for metrics
    - needed for tts configuration via backend
    - copy identity2.json from ovos-skills or enable `skip_auth` in backend config
- (optional) enable backend in ovos-speech
    - [expose ovos-backend to ovos-speech](#expose-ovos-backend-to-other-qubes)
    - needed for metrics
    - needed for wake word configuration via backend
    - needed for ovos-stt-plugin-selene
    - needed for wake word upload
    - needed for utterance upload
    - copy identity2.json from ovos-skills or enable `skip_auth` in backend config
- disable backend in ovos-bus, it's not used
- disable backend in ovos-gui, it's not used
- disable backend in ovos-phal, it's not used

### Expose ovos-bus to other qubes

expose port 8181 from `ovos-bus` to `ovos-XXX` qube

- open a terminal in `dom0`
- `sudo nano /etc/qubes-rpc/policy/qubes.ConnectTCP`
- add a new line with `ovos-XXX @default allow, target=ovos-bus` for every ovos qube
- a reboot will needed for change to take effect

open a terminal in `ovos-XXX` and create the system services to open a socket and connect to ovos-bus on launch

- create bus.socket `sudo nano /rw/config/bus.socket`

```
[Unit]
Description=ovos-bus-service

[Socket]
ListenStream=127.0.0.1:8181
Accept=true

[Install]
WantedBy=sockets.target
```

- create bus.service `sudo nano bus@.service`

```
[Unit]
Description=ovos-bus

[Service]
ExecStart=qrexec-client-vm '' qubes.ConnectTCP+8181
StandardInput=socket
StandardOutput=inherit
```

- edit rc.local to launch the service on qube launch `sudo /rw/config/rc.local `

```
#!/bin/sh

# This script will be executed at every VM startup, you can place your own
# custom commands here. This includes overriding some configuration in /etc,
# starting services etc.
cp -r /rw/config/bus.socket /rw/config/bus@.service /lib/systemd/system/
systemctl daemon-reload
systemctl start bus.socket 
```

### Expose ovos-backend to other qubes

expose port 6712 from `ovos-backend` to `ovos-XXX` qube

- open a terminal in `dom0`
- `sudo nano /etc/qubes-rpc/policy/qubes.ConnectTCP`
- add a new line with `ovos-XXX @default allow, target=ovos-backend` for every ovos qube
- a reboot will needed for change to take effect

open a terminal in `ovos-XXX` and create the system services to open a socket and connect to ovos-backend on launch

- create backend.socket `sudo nano /rw/config/backend.socket`

```
[Unit]
Description=ovos-backend-service

[Socket]
ListenStream=127.0.0.1:6712
Accept=true

[Install]
WantedBy=sockets.target
```

- create backend.service `sudo nano backend@.service`

```
[Unit]
Description=ovos-backend

[Service]
ExecStart=qrexec-client-vm '' qubes.ConnectTCP+6712
StandardInput=socket
StandardOutput=inherit
```

- edit rc.local to launch the service on qube launch `sudo /rw/config/rc.local `

```
#!/bin/sh

# This script will be executed at every VM startup, you can place your own
# custom commands here. This includes overriding some configuration in /etc,
# starting services etc.
cp -r /rw/config/backend.socket /rw/config/backend@.service /lib/systemd/system/
systemctl daemon-reload
systemctl start backend.socket 
```

### Expose ovos-gui to other qubes

expose port 18181 from `ovos-gui` to `ovos-XXX` qube

- open a terminal in `dom0`
- `sudo nano /etc/qubes-rpc/policy/qubes.ConnectTCP`
- add a new line with `ovos-XXX @default allow, target=ovos-gui`
- a reboot will needed for change to take effect

open a terminal in `ovos-gui-client` and create the system service to open a socket and connect to ovos-gui on launch

- create gui.socket `sudo nano /rw/config/gui.socket`

```
[Unit]
Description=ovos-gui-service

[Socket]
ListenStream=127.0.0.1:18181
Accept=true

[Install]
WantedBy=sockets.target
```

- create gui.service `sudo nano gui@.service`

```
[Unit]
Description=ovos-gui

[Service]
ExecStart=qrexec-client-vm '' qubes.ConnectTCP+18181
StandardInput=socket
StandardOutput=inherit
```

- edit rc.local to launch the service on qube launch `sudo /rw/config/rc.local `

```
#!/bin/sh

# This script will be executed at every VM startup, you can place your own
# custom commands here. This includes overriding some configuration in /etc,
# starting services etc.
cp -r /rw/config/gui.socket /rw/config/gui@.service /lib/systemd/system/
systemctl daemon-reload
systemctl start gui.socket 
```

# Troubleshooting

## No audio in ubuntu template

There seems to be a [bug](https://github.com/QubesOS/qubes-issues/issues/6306) in the ubuntu template recommended above

To fix this open a terminal in your ubuntu template + standaloneVMs and run

```bash
sudo ln -s /lib/pulse-13.99/modules/module-vchan-sink.so /lib/pulse-13.99.1/modules/module-vchan-sink.so
```
