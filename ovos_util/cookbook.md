# OVOS CookBook

mostly copy paste-able templates for random stuff

- [OVOS CookBook](#ovos-cookbook)
  * [Messagebus](#messagebus)
    + [Clients](#clients)
      - [Local Enclosure](#local-enclosure)
      - [Admin Service](#admin-service)
      - [Live Translator](#live-translator)
      - [Universal chat](#universal-chat)
    + [Event Trackers](#event-trackers)
      - [Gui Tracker](#gui-tracker)
    + [Bus APIs](#bus-apis)
      - [event alert](#event-alert)
      - [send/receive files](#send-receive-files)
      - [IntentService API](#intentservice-api)
      - [wait for core to be ready](#wait-for-core-to-be-ready)
      - [Providing services over the messagebus](#providing-services-over-the-messagebus)
      - [Querying services over the messagebus](#querying-services-over-the-messagebus)
    + [Metrics](#metrics)
      - [Count utterances](#count-utterances)
  * [Configuration](#configuration)
      - [Change wake word](#change-wake-word)
  * [Skill Development](#skill-development)
    + [Skill Utils](#skill-utils)
      - [PrivateSettings](#privatesettings)
      - [killable_intents](#killable-intents)
    + [Skill Templates](#skill-templates)
      - [Passive Skill](#passive-skill)
    + [Language Support](#language-support)
      - [translation utils](#translation-utils)
    + [Backend Utils](#backend-utils)
      - [get remote settings](#get-remote-settings)


## Messagebus

### Clients

#### Local Enclosure

```python
from ovos_utils.sound.alsa import AlsaControl
#from ovos_utils.sound.pulse import PulseAudio
from ovos_utils.log import LOG
from ovos_utils.messagebus import get_mycroft_bus, Message


class MyPretendEnclosure:

    def __init__(self):
        LOG.info('Setting up client to connect to a local mycroft instance')
        self.bus = get_mycroft_bus()
        # None of these are mycroft signals, but you get the point
        self.bus.on("set.volume", self.handle_set_volume)
        self.bus.on("speak.volume", self.handle_speak_volume)
        self.bus.on("mute.volume", self.handle_mute_volume)
        self.bus.on("unmute.volume", self.handle_unmute_volume)
        
        self.alsa = AlsaControl()
        # self.pulse = PulseAudio()
        
    def speak(self, utterance):
        self.bus.emit(Message('speak', data={'utterance': utterance}))
        
    def handle_speak_volume(self, message):
        volume = self.alsa.get_volume()
        self.speak(volume)
        
    def handle_mute_volume(self, message):
        if not self.alsa.is_muted():
            self.alsa.mute()
        
    def handle_unmute_volume(self, message):
        if self.alsa.is_muted():
            self.alsa.unmute()
        
    def handle_set_volume(self, message):
        volume = message.data.get("volume", 50)
        assert 0 <= volume <= 100
        self.alsa.set_volume(volume)
```

#### Admin Service

```python
from ovos_utils.system import system_reboot, system_shutdown, ssh_enable, ssh_disable
from ovos_utils.log import LOG
from ovos_utils.messagebus import get_mycroft_bus, Message


class AdminService:

    def __init__(self):
        LOG.info('Setting up client to connect to a local mycroft instance')
        self.bus = get_mycroft_bus()
        self.bus.on("system.reboot", self.handle_reboot)
        
    def speak(self, utterance):
        LOG.info('Sending speak message...')
        self.bus.emit(Message('speak', data={'utterance': utterance}))
        
    def handle_reboot(self, message):
        self.speak("rebooting")
        system_reboot()
```

#### Live Translator

```python
from ovos_utils.messagebus import get_mycroft_bus, listen_for_message
from ovos_utils import wait_for_exit_signal
from ovos_utils.lang.translate import say_in_language

bus = get_mycroft_bus()

TARGET_LANG = "pt"


def translate(message):
    utterance = message.data["utterance"]
    say_in_language(utterance, lang=TARGET_LANG)  # will play .mp3 directly

listen_for_message("speak", translate, bus=bus)


wait_for_exit_signal()  # wait for ctrl+c

bus.remove_all_listeners("speak")
bus.close()
```

#### Universal chat

```python
from ovos_utils.messagebus import send_message, listen_for_message
from ovos_utils.lang.translate import translate_text
from ovos_utils.lang.detect import detect_lang
from ovos_utils.log import LOG


OUTPUT_LANG = "pt"  # received messages will be in this language
MYCROFT_LANG = "en"  # mycroft is configured in this language


def handle_speak(message):
    utt = message.data["utterance"]
    utt = translate_text(utt, "pt")  # source lang is auto detected
    print("MYCROFT:", utt)


bus = listen_for_message("speak", handle_speak)

print("Write in any language and mycroft will answer in {lang}".format(lang=OUTPUT_LANG))


while True:
    try:
        utt = input("YOU:")
        lang = detect_lang("utt")   # source lang is auto detected, this is optional
        if lang != MYCROFT_LANG:
            utt = translate_text(utt)
        send_message("recognizer_loop:utterance",
                     {"utterances": [utt]},
                     bus=bus # re-utilize the bus connection
                     )
    except KeyboardInterrupt:
        break
    except Exception as e:
        LOG.exception(e)

bus.remove_all_listeners("speak")
bus.close()
```

### Event Trackers

#### Gui Tracker

```python
from ovos_utils.gui import GUITracker
from ovos_utils import wait_for_exit_signal


class MyGUIEventTracker(GUITracker):
    # GUI event handlers
    # user can/should subclass this
    def on_idle(self, namespace):
        print("IDLE", namespace)
        timestamp = self.idle_ts

    def on_active(self, namespace):
        # NOTE: page has not been loaded yet
        # event will fire right after this one
        print("ACTIVE", namespace)
        # check namespace values, they should all be set before this event
        values = self.gui_values[namespace]

    def on_new_page(self, page, namespace, index):
        print("NEW PAGE", namespace, index, namespace)
        # check all loaded pages
        for n in self.gui_pages:  # list of named tuples
            nspace = n.name  # namespace / skill_id
            pages = n.pages  # ordered list of page uris

    def on_gui_value(self, namespace, key, value):
        # WARNING this will pollute logs quite a lot, and you will get
        # duplicates, better to check values on a different event,
        # demonstrated in on_active
        print("VALUE", namespace, key, value)


g = MyGUIEventTracker()

print("device has screen:", g.can_display())
print("mycroft-gui installed:", g.is_gui_installed())
print("gui connected:", g.is_gui_connected())


# check registered idle screens
print("Registered idle screens:")
for name in g.idle_screens:
    namespace = g.idle_screens[name]
    print("   - ", name, ":", namespace)


# just block listening for events until ctrl + C
wait_for_exit_signal()
```

### Bus APIs

#### event alert

```python
from ovos_utils.messagebus import send_message
from ovos_utils.log import LOG
from ovos_utils import create_daemon, wait_for_exit_signal
import random
from time import sleep


def alert():
    LOG.info("Alerting user of some event using Mycroft")
    send_message("speak", {"utterance": "Alert! something happened"})


def did_something_happen():
    while True:
        if random.choice([True, False]):
            alert()
        sleep(10)


create_daemon(did_something_happen) # check for something in background
wait_for_exit_signal()  # wait for ctrl+c
```

#### send/receive files

```python
from ovos_utils.messagebus import send_binary_data_message, \
    send_binary_file_message, decode_binary_message, listen_for_message

from time import sleep
import json
from os.path import dirname, join

# random file
my_file_path = join(dirname(__file__), "music.txt")
with open(my_file_path, "rb") as f:
    original_binary = f.read()


def receive_file(message):
    print("Receiving file")
    path = message.data["path"]
    print(path)
    binary_data = decode_binary_message(message)


listen_for_message("mycroft.binary.file", receive_file)
sleep(1)
send_binary_file_message(my_file_path)


def receive_binary(message):
    print("Receiving binary data")
    binary_data = decode_binary_message(message)
    print(binary_data == original_binary)


listen_for_message("mycroft.binary.data", receive_binary)
sleep(1)
send_binary_data_message(original_binary)
```

#### IntentService API

```python
from ovos_utils.intents import IntentQueryApi
from pprint import pprint


intents = IntentQueryApi()

pprint(intents.get_skill("who are you"))
pprint(intents.get_skill("set volume to 100%"))

# loaded skills
pprint(intents.get_skills_manifest())
pprint(intents.get_active_skills())

# intent parsing
pprint(intents.get_adapt_intent("who are you"))
pprint(intents.get_padatious_intent("who are you"))
pprint(intents.get_intent("who are you"))  # intent that will trigger

# skill from utterance
pprint(intents.get_skill("who are you"))

# registered intents
pprint(intents.get_adapt_manifest())
pprint(intents.get_padatious_manifest())
pprint(intents.get_intent_manifest())  # all of the above

# registered vocab
pprint(intents.get_entities_manifest())  # padatious entities / .entity files
pprint(intents.get_vocab_manifest())  # adapt vocab / .voc files
pprint(intents.get_regex_manifest())  # adapt regex / .rx files
pprint(intents.get_keywords_manifest())  # all of the above
```

#### wait for core to be ready

```python
from ovos_utils.skills import skills_loaded
from time import sleep

loaded = False
while not loaded:
    loaded = skills_loaded()
    sleep(0.5)

print("Skills all loaded!")
```

#### Providing services over the messagebus

```python
from ovos_utils.messagebus import BusFeedProvider, Message
from datetime import datetime


class ClockService(BusFeedProvider):
    def __init__(self, name="clock_transmitter", bus=None):
        trigger_message  = Message("time.request")
        super().__init__(trigger_message, name, bus)
        self.set_data_gatherer(self.handle_get_time)
        
    def handle_get_time(self, message):
        self.update({"date": datetime.now()})
        
        
clock_service = ClockService()
```

#### Querying services over the messagebus

```python
from ovos_utils.messagebus import BusFeedConsumer, Message

class Clock(BusFeedConsumer):
    def __init__(self, name="clock_receiver", timeout=3, bus=None):
        request_message = Message("time.request")
        super().__init__(request_message, name, timeout, bus)

clock = Clock()
date = clock.request()["date"]
```

### Metrics

#### Count utterances

```python
from ovos_utils.messagebus import listen_for_message
from ovos_utils.log import LOG
from ovos_utils import wait_for_exit_signal

heard = 0
spoken = 0


def handle_speak(message):
    global spoken
    spoken += 1
    LOG.info("Mycroft spoke {n} sentences since start".format(n=spoken))


def handle_hear(message):
    global heard
    heard += 1
    LOG.info("Mycroft responded to {n} sentences since start".format(n=heard))


bus = listen_for_message("speak", handle_speak)
listen_for_message("recognize_loop:utterance", handle_hear, bus=bus)  # re utilize bus

wait_for_exit_signal()  # wait for ctrl+c

# cleanup is a good practice!
bus.remove_all_listeners("speak")
bus.remove_all_listeners("recognize_loop:utterance")
bus.close()
```

## Configuration

#### Change wake word

```python
from ovos_utils.configuration import update_mycroft_config
from ovos_utils.lang.phonemes import get_phonemes


def create_wakeword(word, sensitivity):
    # sensitivity is a bitch to do automatically
    # TODO make some web ui or whatever to tweak it experimentally
    phonemes = get_phonemes(word)
    config = {
        "listener": {
            "wake_word": word
        },
        word: {
            "andromeda": {
                "module": "pocketsphinx",
                "phonemes": phonemes,
                "sample_rate": 16000,
                "threshold": sensitivity,
                "lang": "en-us"
            }
        }
    }
    update_mycroft_config(config)


create_wakeword("andromeda", "1e-25")
```


## Skill Development

### Skill Utils

#### PrivateSettings

```python
from ovos_utils.skills.settings import PrivateSettings


with PrivateSettings("testskill.jarbasai") as settings:
    print(settings.path)  # ~/.cache/json_database/testskill.jarbasai.json
    settings["key"] = "value"

    meta = settings.settingsmeta
    # can be used for displaying in GUI
    """
    {'skillMetadata': {'sections': [{'fields': [{'label': 'Key',
                                             'name': 'key',
                                             'type': 'text',
                                             'value': 'value'}],
                                 'name': 'testskill.jarbasai'}]}}
    """

    # auto saved when leaving "with" context
    # you can also manually call settings.store() if not using "with" context
```

#### killable_intents

```python
from ovos_utils.waiting_for_mycroft.base_skill import killable_intent, MycroftSkill
from mycroft import intent_file_handler
from time import sleep


class Test(MycroftSkill):
    """
    send "mycroft.skills.abort_question" and confirm only get_response is aborted
    send "mycroft.skills.abort_execution" and confirm the full intent is aborted, except intent3
    send "my.own.abort.msg" and confirm intent3 is aborted
    say "stop" and confirm all intents are aborted
    """
    def __init__(self):
        super(Test, self).__init__("KillableSkill")
        self.my_special_var = "default"

    def handle_intent_aborted(self):
        self.speak("I am dead")
        # handle any cleanup the skill might need, since intent was killed
        # at an arbitrary place of code execution some variables etc. might
        # end up in unexpected states
        self.my_special_var = "default"

    @killable_intent(callback=handle_intent_aborted)
    @intent_file_handler("test.intent")
    def handle_test_abort_intent(self, message):
        self.my_special_var = "changed"
        while True:
            sleep(1)
            self.speak("still here")

    @intent_file_handler("test2.intent")
    @killable_intent(callback=handle_intent_aborted)
    def handle_test_get_response_intent(self, message):
        self.my_special_var = "CHANGED"
        ans = self.get_response("question", num_retries=99999)
        self.log.debug("get_response returned: " + str(ans))
        if ans is None:
            self.speak("question aborted")

    @killable_intent(msg="my.own.abort.msg", callback=handle_intent_aborted)
    @intent_file_handler("test3.intent")
    def handle_test_msg_intent(self, message):
        if self.my_special_var != "default":
            self.speak("someone forgot to cleanup")
        while True:
            sleep(1)
            self.speak("you can't abort me")


def create_skill():
    return Test()

```

[source_code](https://github.com/OpenVoiceOS/skill-abort-test)

### Skill Templates

#### Passive Skill

Listen to all utterances

```python
from ovos_utils.skills.templates.passive import PassiveSkill

class MySkill(PassiveSkill):

    def handle_utterance(self, utterances, lang="en-us"):
        """ Listen to all utterances passively, eg, take metrics """
```
https://github.com/OpenVoiceOS/ovos_utils/blob/alpha/0.0.8/ovos_utils/skills/templates/passive.py

### Language Support


#### translation utils

```python
from ovos_utils.lang import detect_lang, translate_text

# detecting language
detect_lang("olá eu chamo-me joaquim")  # "pt"
detect_lang("olá eu chamo-me joaquim", return_dict=True)
"""{'confidence': 0.9999939001351439, 'language': 'pt'}"""

detect_lang("hello world")  # "en"
detect_lang("This piece of text is in English. Този текст е на Български.", return_dict=True)
"""{'confidence': 0.28571342657428966, 'language': 'en'}"""

# translating text
#  - source lang will be auto detected using utils above
#  - default target language is english
translate_text("olá eu chamo-me joaquim")
"""Hello I call myself joaquim"""

#  - you should specify source lang whenever possible to save 1 api call
translate_text("olá eu chamo-me joaquim", source_lang="pt", lang="es")
"""Hola, me llamo Joaquim"""
```

### Backend Utils

#### get remote settings

```python
from ovos_utils.skills.settings import get_all_remote_settings, get_remote_settings

""" 
WARNING: selene backend does not use a proper skill_id, if you have
skills with same name but different author settings will overwrite each
other on the backend

2 way sync with Selene is also not implemented and causes a lot of issues, 
these problems have been reported over 9 months ago (and counting)

I strongly recommend you validate returned data as much as possible, 
maybe ask the user to confirm any action, DO NOT automate settings sync

skill matching is currently done by checking "if {skill} in string"
once mycroft fixes it on their side this will start using a proper
unique identifier

THIS METHOD IS NOT ALWAYS SAFE
"""

# get raw skill settings payload
all_remote_settings = get_all_remote_settings()
"""
{'@0e813c09-8dbc-40ed-974c-5705274a7b55|mycroft-npr-news|20.08': {'custom_url': '',
                                                                  'station': 'not_set',
                                                                  'use_curl': True},
 '@0e813c09-8dbc-40ed-974c-5705274a7b55|mycroft-pairing|20.08': None,
 '@0e813c09-8dbc-40ed-974c-5705274a7b55|mycroft-volume|20.08': {'ducking': True},
 (...)
 'mycroft-weather|20.08': {'units': 'default'},
 'mycroft-wiki|20.08': None}
"""

# search remote settings for a specific skill
voip_remote_settings = get_remote_settings("skill-voip")
"""
{'add_contact': False,
 'auto_answer': True,
 'auto_reject': False,
 'auto_speech': 'I am busy, call again later',
 'contact_address': 'user@sipxcom.com',
 'contact_name': 'name here',
 'delete_contact': False,
 'gateway': 'sipxcom.com',
 'password': 'SECRET',
 'sipxcom_gateway': 'https://sipx.mattkeys.net',
 'sipxcom_password': 'secret',
 'sipxcom_sync': False,
 'sipxcom_user': 'MattKeys',
 'user': 'user'}
"""
```
