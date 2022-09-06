# ovos-qubes
setting up a hardened ovos-core under QubesOS


## architecture

- sys-ovos-firewall 
  - DispVM
  - set firewall rules
  - can be connected to a VPN
  - can use mirage firewall unikernel
  - all networked ovos qubes connect here
- template-ovos-base
  - templateVM
  - /etc/mycroft/mycroft.conf lives here
  - system packages installed/updated here
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
  - based on ubuntu template
  - mycroft-gui installed as user from source
  - sys-ovos-firewall needed for stream playback / web browsing / remote pictures

  
## template-ovos-base

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
  "stt": {"module": "ovos-stt-plugin-server"},
  "tts": {"module": "ovos-tts-plugin-mimic3"},
  "padatious": {"regex_only": true},
  "debug": true,
  "log_level": "DEBUG"
}
```
