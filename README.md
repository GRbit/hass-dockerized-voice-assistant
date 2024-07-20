# hass-dockerized-voice-assistant
A docker-compose for starting home assistant with other services necessary for working voice assistant. Just trun
`docker-compose up -d` and you'll have everything that is required to enable and use voice assistant in HA using **speaker
and microphone available on your server**.

## Prepace

On the one hand, this repository might seem rather simple and not worth sharing, while on the other hand, I see the HomeAssistant
(HA) community lacks support for people using Docker. All I can see in the relevant [forum
topics](https://community.home-assistant.io/t/addons-for-docker-installation/436190) is "Docker is very complicated,
it's for very advanced users, if you want add-ons, go with supervised and HAOS installations".

So, in my opinion, Docker is super-duper simple. That's why it's so popular. Ok, maybe it would be a bit hard for
for your grandma to understand and manage, but it's not that hard for anyone with a little bit of IT knowledge.
actually. And if I have a small home server that manages my photos, storage, music and many other services, there is no
I install some HAOS on it, and I don't even want to install any supervisors. People usually have their own
own ways of controlling it, backing it up and managing everything (in my case it's simple - put everything in Docker).

The real problem is that all these add-ons that are only available for supervised installations are poorly documented, if at all.
documented at all. Even containers that mention some API have no guides on how to communicate with them. If you try to do
simple things, you have no chance of actually finding solutions. I spent many hours trying to access my microphone from
my microphone from Home Assistant. Not any wifi/bluetooth device or smart speaker, I just have an old Jabra Speak, why
Why the hell can't I use it as soon as I connect it to the server? You can try to install it on your Docker
Home Assistant installation, good luck finding a solution.

So, after finally getting everything set up for the voice assistant, I decided that, given how many hours I've spent on it
should at least be documented like a log. And I'd be happy if it saved some time for any other Docker HA
enthusiasts who want a voice assistant.

## What's inside?

There are several things (number is for ports they are working on):

* homeassistant 8123 – that's the obvious part
* mosquitto 1883 and 9001 – not required for voice, but very useful for other things
* openwakeword 10400 — required to wake your voice assistant up
* faster-whisper 10300 — required to transcribe your speach into words and pass them to HA
* piper 10200 — synthesize the answer from HA
* satellite 10700 – connect local mice and speaker via Wyoming protocol to HA (that's the piece I've been looking for
  very long)
* playback 10601 – play the sound on your server's speaker (only connected to satellite)
* microphone 10600 – records the sound on your server's microphone (only connected to stallite)

#### Wath's important to know?

First of all, playbak and microphone configs need to point to the sound sink/source that fits your server (look at the
command section of the corresponding containers). To list all available decies run `arecord -L` or `aplay -L` and test
it to find a suitable option.

I've added `CONTAINERS_CONFIGS` variable in `.env` file. You can use it as folder to put most of the configs that you
can edit (or to copy configs of your existing HomeAssistant installation for example).

It's important to know that most of the services are availbale are available for everyone seing your server, so when
you'll be settings up vioce assistant with 
[this tutorial](https://www.home-assistant.io/voice_control/voice_remote_local_assistant/), you can use either something
like `192.168.0.2:10200` or `piper:10200` and it'll work.

#### Inetersting to know

It's hard to find information on how to actually interact via Wyoming protocol with all these services. There is a link
on Home Assistant website to some [nice pcitures and description](https://github.com/rhasspy/rhasspy3/blob/master/docs/wyoming.md)
but they are generalized (which makes total sense for a protocol). When you go to the [protocol repository](https://github.com/rhasspy/wyoming)
you'll find more information. Now you know something about the commands the server can handle, but it's still hard to
understand the exact format. Should you put your `synthesize` request in `data` or add it as an "Additional data" on the
next line? You can't tell untill you go through the code (or in my case catch the packets with wireshark).
