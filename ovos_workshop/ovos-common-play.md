# About

TODO how does it work

## MediaTypes

TODO

## Skills

TODO

# Playback options

a skill can return results that require different capabilities

TODO

## Audio Service

the mycroft audio service handles playback, ovos common play syncs the GUI with the external player

![](https://github.com/OpenVoiceOS/OVOS-workshop/raw/master/screenshots/audio_service.png)

if track length is not reported it is assumed to be a livestream and seekbar is not displayed

notice that bandcamp did not report track length, in the future this info will be retrieved from the external player when possible

![](https://github.com/OpenVoiceOS/OVOS-workshop/raw/master/screenshots/missing_seekbar.png)


## GUI Player

Gui player is a dedicated media playback plugin, it technically uses the audio service but the playback is handled directly in the GUI

https://github.com/OpenVoiceOS/ovos-guiplayer-plugin
https://github.com/MycroftAI/plugin-playback-gui-player

TODO

### GUI (simple video)

A configurable alternative for a simpler video player, for those who want a minimalist look

![](https://github.com/OpenVoiceOS/OVOS-workshop/raw/master/screenshots/simple_video.png)

