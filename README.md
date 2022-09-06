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
  + [ovos-gui](#ovos-gui)
  + [ovos-gui-client](#ovos-gui-client)
* [Connecting the Qubes](#connecting-the-qubes)
  + [Exposing port 8181 from ovos-bus](#exposing-port-8181-from-ovos-bus)
  + [Exposing port 6712 from ovos-backend](#exposing-port-6712-from-ovos-backend)
  + [Exposing port 18181 from ovos-gui](#exposing-port-18181-from-ovos-gui)
  + [Create ovos-bus system services](#create-ovos-bus-system-services)
  + [Create ovos-backend system services](#create-ovos-backend-system-services)
  + [Create ovos-gui system service](#create-ovos-gui-system-service)

## Architecture

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
- ovos-gui-client
  - StandaloneVM
  - based on [ubuntu template](https://qubes.3isec.org/Templates_4.1)
  - mycroft-gui installed as user from source
  - sys-ovos-firewall needed for stream playback / web browsing / remote pictures

## Creating OVOS Qubes
  
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
    "stt": {"module": "ovos-stt-plugin-selene"},
    "tts": {"module": "ovos-tts-plugin-mimic3"},
    "padatious": {"regex_only": true},
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
    "opt_in": true,
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
    "backend_port": 6712,
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
- create auto_start.sh `nano auto-start.sh`
  ```bash
  /home/user/.local/bin/ovos-local-backend >> /home/user/backend.log 2>&1 &
  ```
- create .desktop file to autostart ovos-backend when the VM boots `nano /home/user/.config/autostart/ovos-backend.desktop`
  ```
  [Desktop Entry]
  Name=OVOS Local Backend
  Exec=bash /home/user/auto-start.sh
  Type=Application
  StartupNotify=true
  NoDisplay=true
  ```
- configure mycroft.conf in `template-ovos-base` or in every independent `ovos-XXX` qube
  ```
  {
    "server": {
      "url": "http://0.0.0.0:6712",
      "version": "v1",
      "update": true,
      "metrics": true
    },
    "listener": {
      "wake_word_upload": {
        "url": "http://0.0.0.0:6712/precise/upload"
      }
    }
  }
  ```
  
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
- create .desktop file to autostart ovos-bus when the VM boots `nano /home/user/.config/autostart/mycroft-messagebus.desktop`
  ```
  [Desktop Entry]
  Name=OVOS Messagebus
  Exec=bash /home/user/auto-start.sh
  Type=Application
  StartupNotify=true
  NoDisplay=true
  ```
- (optional) expose ovos-backend to ovos-bus (see below)

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
- create .desktop file to autostart ovos-audio when the VM boots `nano /home/user/.config/autostart/mycroft-audio.desktop`
  ```
  [Desktop Entry]
  Name=OVOS Audio Service
  Exec=bash /home/user/auto-start.sh
  Type=Application
  StartupNotify=true
  NoDisplay=true
  ```
- expose ovos-bus to ovos-audio (see below)
- (optional) expose ovos-backend to ovos-audio (see below)
  - needs to be set in mycroft.conf
  - needed for metrics (opt in)
  
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
- create .desktop file to autostart ovos-skills when the VM boots `nano /home/user/.config/autostart/mycroft-skills.desktop`
  ```
  [Desktop Entry]
  Name=OVOS Skills Service
  Exec=bash /home/user/auto-start.sh
  Type=Application
  StartupNotify=true
  NoDisplay=true
  ```
- expose ovos-bus to ovos-skills (see below)
- (optional) expose ovos-backend to ovos-skills (see below)
  - needs to be set in mycroft.conf
  - needed for some skills
  - needed for pairing
  - needed for metrics (opt in)
  
### ovos-speech
- create `ovos-speech` qubes from `template-ovos-base`
- (optional) select `sys-ovos-firewall` as NetVM
  - not needed if you setup an offline STT plugin 
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
- create .desktop file to autostart ovos-speech when the VM boots `nano /home/user/.config/autostart/mycroft-stt.desktop`
  ```
  [Desktop Entry]
  Name=OVOS Speech Client
  Exec=bash /home/user/auto-start.sh
  Type=Application
  StartupNotify=true
  NoDisplay=true
  ```
- expose ovos-bus to ovos-speech (see below)
- (optional) expose ovos-backend to ovos-speech (see below)
  - needs to be set in mycroft.conf
  - integrates with selene stt plugin
  - needed for metrics (opt in)
  - needed for wake word upload (opt in)
- [attach microphone](https://www.qubes-os.org/doc/how-to-use-devices/#attaching-devices)

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
- create .desktop file to autostart ovos-gui when the VM boots `nano /home/user/.config/autostart/mycroft-gui-messagebus.desktop`
  ```
  [Desktop Entry]
  Name=OVOS GUI Messagebus
  Exec=bash /home/user/auto-start.sh
  Type=Application
  StartupNotify=true
  NoDisplay=true
  ```
- expose ovos-bus to ovos-gui (see below)
- (optional) expose ovos-backend to ovos-gui (see below)

### ovos-gui-client
- install [community](https://www.qubes-os.org/doc/templates/#community) ubuntu template in dom0
  - download [here](https://qubes.3isec.org/Templates_4.1) to some qube, eg `dl_qube`
  - copy downloaded file to dom0 `qvm-run --pass-io dl_qube 'cat /home/user/Downloads/focal.rpm' > /home/user/focal.rpm`
  - install `sudo dnf install focal.rpm`
- create `ovos-gui-client` qubes from `focal` as a StandaloneVM
- select `sys-ovos-firewall` as NetVM
- install mycroft-gui
  ```bash
  git clone https://github.com/MycroftAI/mycroft-gui
  cd mycroft-gui
  ./dev_setup.sh
  ```
- expose ovos-bus to ovos-gui-client (see below)
- expose ovos-gui to ovos-gui-client (see below)
- launch `mycroft-gui-app` from this qube when wanted
  - (optional) launch on qube on boot
  - (optional) launch mycroft gui on VM startup

## Connecting the Qubes

We need to [open a TCP port to other network-isolated qubes](https://www.qubes-os.org/doc/firewall/#opening-a-single-tcp-port-to-other-network-isolated-qube) for ovos-bus, ovos-gui and ovos-backend

### Exposing port 8181 from ovos-bus

expose port 8181 from `ovos-bus` to all `ovos-XXX` qubes

- open a terminal in `dom0`
- `sudo nano /etc/qubes-rpc/policy/qubes.ConnectTCP`
- add a new line with `ovos-XXX @default allow, target=ovos-bus` for every ovos qube
- a reboot will needed for change to take effect

### Exposing port 6712 from ovos-backend

expose port 6712 from `ovos-backend` to all `ovos-XXX` qubes

- open a terminal in `dom0`
- `sudo nano /etc/qubes-rpc/policy/qubes.ConnectTCP`
- add a new line with `ovos-XXX @default allow, target=ovos-backend` for every ovos qube
- a reboot will needed for change to take effect


### Exposing port 18181 from ovos-gui

expose port 18181 from `ovos-gui` to `ovos-gui-client` qube

- open a terminal in `dom0`
- `sudo nano /etc/qubes-rpc/policy/qubes.ConnectTCP`
- add a new line with `ovos-gui-client @default allow, target=ovos-gui`
- a reboot will needed for change to take effect


### Create ovos-bus system services

open a terminal in `ovos-XXX` and create the system services to connect to ovos-bus on launch

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

### Create ovos-backend system services

open a terminal in `ovos-XXX` and create the system services to connect to ovos-backend on launch

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


### Create ovos-gui system service

open a terminal in `ovos-gui-client` and create the system service to connect to ovos-gui on launch

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
