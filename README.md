# hass-dockerized-voice-assistant
A docker compose setup for starting Home Assistant with other services necessary for a working voice assistant. Just run
`docker compose up -d` and you'll have everything that is required to enable and use voice assistant in HA using
**speaker and microphone available on your server**.

> **Note: this repository is somewhat outdated.** It documents the Wyoming-based setup as I built it in 2024.
> Since then [rhasspy/wyoming-satellite](https://github.com/rhasspy/wyoming-satellite) (this compose builds a fork of
> it), [rhasspy/wyoming-mic-external](https://github.com/rhasspy/wyoming-mic-external) and
> [rhasspy/wyoming-snd-external](https://github.com/rhasspy/wyoming-snd-external) have been archived. The
> wyoming-satellite README says it was replaced by
> [Linux Voice Assistant](https://github.com/OHF-Voice/linux-voice-assistant), which uses the ESPHome protocol and
> supports newer features (media player, stop wake word, start/continue conversation, timers). If you are starting
> from scratch, look at Linux Voice Assistant first.

## Preface

On the one hand, this repository might seem rather simple and not worth sharing, while on the other hand, I see the
HomeAssistant (HA) community lacks support for people using Docker. All I can see in the relevant [forum
topics](https://community.home-assistant.io/t/addons-for-docker-installation/436190) is "Docker is very complicated,
it's for very advanced users, if you want add-ons, go with supervised and HAOS installations".

So, in my opinion, Docker is super-duper simple. That's why it's so popular. Ok, maybe it would be a bit hard for your
grandma to understand and manage, but it's not that hard for anyone with a little bit of IT knowledge, actually. And if
I have a small home server that manages my photos, storage, music and many other services, there is no way I'll
install some HAOS on it, and I don't even want to install any supervisors. People usually have their own ways of
controlling it, backing it up and managing everything (in my case it's simple - put everything in Docker).

The real problem is that all these add-ons that are only available for supervised installations are poorly
documented, if at all. Even containers that mention some API have no guides on how to communicate with them. If you try
to do simple things, you have no chance of actually finding solutions. I spent many hours trying to access my
microphone from Home Assistant. Not any wifi/bluetooth device or smart speaker, I just have an old Jabra Speak. Why the
hell can't I use it as soon as I connect it to the server? You can try to install it on your Docker Home Assistant
installation, good luck finding a solution.

So, after finally getting everything set up for the voice assistant, I decided that, given how many hours I've spent
on it, it should at least be documented like a log. And I'd be happy if it saved some time for any other Docker HA
enthusiasts who want a voice assistant.

## What's inside?

There are several things (number is for ports they are working on):

* homeassistant 8123 - that's the obvious part (runs on the host network)
* mosquitto 1883 and 9001 - not required for voice, but very useful for other things (open to your LAN)
* openwakeword 10400 - required to wake your voice assistant up
* faster-whisper 10300 - required to transcribe your speech into words and pass them to HA
* piper 10200 - synthesize the answer from HA
* satellite 10700 - connect local mic and speaker via Wyoming protocol to HA (that's the piece I've been looking for
  very long)
* playback 10601 - play the sound on your server's speaker (only connected to satellite)
* microphone 10600 - records the sound on your server's microphone (only connected to satellite)

Ports 10200, 10300, 10400 and 10700 are published on `127.0.0.1` only, so they are reachable only from the server
itself: Wyoming has no authentication, and anyone who can reach these ports can use them. Playback and microphone are
not published at all, the satellite talks to them (and to openwakeword) by service name over the compose network.

## Step by step

### Prerequisites

* Docker with the compose plugin (`docker compose version` should work).
* A USB microphone and speaker (or a speakerphone like my Jabra) connected to the server.
* `aplay` and `arecord` on the host (the `alsa-utils` package on most distros) to find and test the audio device.

### Steps

1. Clone the repository and go into it:

   ```sh
   git clone https://github.com/GRbit/hass-dockerized-voice-assistant.git
   cd hass-dockerized-voice-assistant
   ```

2. Create your `.env` from the example and edit it:

   ```sh
   cp .env.example .env
   ```

   * `CONTAINERS_CONFIGS` - folder where most of the configs live. You can edit them there (or copy configs of your
     existing HomeAssistant installation into it, for example).
   * `PUID` / `PGID` - the host user that owns that folder (run `id` to find yours). The linuxserver.io images
     (homeassistant, faster-whisper, piper) start as root and drop to this user; the others are started directly as
     this user.
   * `SND_DEVICE` / `MIC_DEVICE` - the sound sink/source that fits your server, see step 4.

3. Create the config directories owned by that user and copy the mosquitto config:

   ```sh
   set -a; . ./.env; set +a
   sudo install -d -o "$PUID" -g "$PGID" \
     "$CONTAINERS_CONFIGS"/homeassistant/backup \
     "$CONTAINERS_CONFIGS"/homeassistant/config \
     "$CONTAINERS_CONFIGS"/mosquitto/config \
     "$CONTAINERS_CONFIGS"/mosquitto/data \
     "$CONTAINERS_CONFIGS"/mosquitto/log \
     "$CONTAINERS_CONFIGS"/openwakeword \
     "$CONTAINERS_CONFIGS"/faster-whisper/config \
     "$CONTAINERS_CONFIGS"/piper/config
   sudo install -o "$PUID" -g "$PGID" -m 644 mosquitto/config/mosquitto.conf "$CONTAINERS_CONFIGS"/mosquitto/config/
   ```

   The mosquitto container mounts the whole `$CONTAINERS_CONFIGS/mosquitto` folder over `/mosquitto`, which hides the
   config built into the image, so without this copy the broker has no config. The shipped one allows anonymous
   clients, which is fine on a LAN; set up a password file before exposing it anywhere else.

4. Find your audio device. To list all available devices run `aplay -L` (playback) and `arecord -L` (microphone),
   then test the one you picked. The microphone container records with
   `arecord -D <device> -r 16000 -c 1 -f S16_LE -t raw`, so test with the same parameters (as wav, so you can play it
   back):

   ```sh
   arecord -D "$MIC_DEVICE" -r 16000 -c 1 -f S16_LE -d 5 test.wav
   aplay -D "$SND_DEVICE" test.wav
   ```

   If you hear yourself, put these device names into `SND_DEVICE` and `MIC_DEVICE` in `.env` (re-run the
   `set -a; . ./.env; set +a` line after editing it).

5. Start everything:

   ```sh
   docker compose up -d
   ```

   Home Assistant runs on the host network (the linuxserver.io README recommends it so HA can discover devices on
   your network), and the UI is at `http://<server>:8123` as usual. Follow the onboarding wizard there.

6. Add the voice services to Home Assistant. Go to **Settings > Devices & services**, select **Add Integration**,
   pick **Wyoming Protocol** and enter host and port. Repeat it for every service:

   | Service        | Host        | Port  |
   |----------------|-------------|-------|
   | faster-whisper | `127.0.0.1` | 10300 |
   | piper          | `127.0.0.1` | 10200 |
   | openwakeword   | `127.0.0.1` | 10400 |
   | satellite      | `127.0.0.1` | 10700 |

   Use `127.0.0.1`, not the service names or your server's LAN IP: because HA is on the host network, it is not on
   the compose network and can't resolve names like `piper`, and the ports are bound to loopback only. Some services
   may already show up as **Discovered**, then you can just confirm them.

   If you want MQTT, add the **MQTT** integration the same way with broker `127.0.0.1` and port `1883`.

7. Create an Assist pipeline, as in [this tutorial](https://www.home-assistant.io/voice_control/voice_remote_local_assistant/).
   Go to **Settings > Voice assistants** and select **Add assistant**. Give it a name, pick the language, then under
   **Conversation agent** select **Home Assistant**, under **Speech-to-text** select faster-whisper and under
   **Text-to-speech** select Piper. Then open the satellite device (**Settings > Devices & services > Wyoming
   Protocol**) and choose this pipeline in its **Assistant** select.

8. Test it: say "ok nabu" and a command. You should hear the awake sound, then the done sound and the answer. If
   nothing happens, `docker compose logs -f satellite microphone playback openwakeword` shows where it gets stuck
   (all of them run with `--debug`).

## Interesting to know

It's hard to find information on how to actually interact via Wyoming protocol with all these services. There is a link
on Home Assistant website to some [nice pictures and description](https://github.com/rhasspy/rhasspy3/blob/master/docs/wyoming.md)
but they are generalized (which makes total sense for a protocol). When you go to the [protocol repository](https://github.com/rhasspy/wyoming)
you'll find more information. Now you know something about the commands the server can handle, but it's still hard to
understand the exact format. Should you put your `synthesize` request in `data` or add it as an "Additional data" on the
next line? You can't tell until you go through the code (or in my case catch the packets with wireshark).
