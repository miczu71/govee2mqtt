# Govee to MQTT bridge: Home Assistant Add-On (miczu71 fork)

This is a fork of [wez/govee2mqtt](https://github.com/wez/govee2mqtt) that retains the
`gv2mqtt/availability` MQTT topic (both the "online" publish and the "offline" last will),
so that entities don't get stuck `unavailable` when Home Assistant is slow to (re)subscribe
around a Core restart. See
[`docs/ROADMAP-retain-availability.md`](https://github.com/miczu71/govee2mqtt/blob/main/docs/ROADMAP-retain-availability.md)
for the root-cause writeup. Track upstream for when/if this lands there: wez/govee2mqtt#110,
wez/govee2mqtt#572.

This addon provides a bridge between [Govee](https://govee.com) lights and Home Assistant,
via the [Home Assistant MQTT Integration](https://www.home-assistant.io/integrations/mqtt/).

