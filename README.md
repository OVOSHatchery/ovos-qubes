# ovos-qubes
setting up a hardened ovos-core under QubesOS


## architecture

- sys-ovos-firewall 
  - DispVM
  - set firewall rules
  - can be connected to a VPN
  - can use mirage firewall unikernel
  - all networked ovos qubes connect here
- ovos-base-template
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

  
## ovos-base-template

- clone a debian template VM as `template-ovos-base`
- debloat the image

- install packages shared across all VMs
