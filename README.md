geoclue provider snap (Ubuntu Core Desktop, Phase 4)
====================================================

The fat base shipped geoclue (location service); the thin desktop
variant bakes NOTHING geoclue (no config, no users beyond the users
wave's geoclue uid 203, no dbus/policy files). This snap restores the
daemon side: geoclue-2.0 2.7.2 staged from the resolute archive
(byte-identical to the fat base's daemon), as a strictly-confined
system dbus-activated daemon (org.freedesktop.GeoClue2). The demo
agent stays unshipped — the fat base disabled it (core-base-desktop
commit e140d44) and gnome-shell is the real agent.

Design (follows the fprintd/accountsservice provider-snap pattern):

- base: plain core26 — the daemon execs no host helpers; all its
  libraries (soup3, mm-glib, avahi, json-glib, ...) stage with the
  deb; no localdev_swap_base dance.
- slots: `geoclue-dbus` (builtin dbus interface, system bus,
  org.freedesktop.GeoClue2 — activation + name ownership) and
  `geoclue` (custom fork interface carrying the daemon recipe:
  peer-less dbus receive rules for the location API, peer-less sends
  into the session agents and the data sources — wpa_supplicant,
  ModemManager, Avahi —, /etc/geoclue reads, network inet/inet6
  stream/dgram for the NMEA UDP listener and GeoIP lookups; the
  connected plug also carries the AGENT receive rules so the shell
  can receive Authorize calls).
- plugs: `network` (data sources). NO polkit plug (geoclue has no
  polkit actions).
- apps: `geoclue` daemon (daemon: dbus, daemon-scope: system,
  activates-on geoclue-dbus) with the fat unit's
  GSETTINGS_BACKEND=memory environment.
- layout: /etc/geoclue -> $SNAP/etc/geoclue (the variant bakes no
  geoclue config, so the snap ships the fat base's own geoclue.conf
  + conf.d/ staged from the identical resolute deb). /etc/geolocation
  (static source) is granted but absent, as on the fat base.
- libgeoclue-2-0 (client library) is deliberately NOT staged here;
  libgeoclue clients link it from the content snap.

Build: ./go-build (needs an apt-cacher/proxy env from go-build-lib.sh).

BEHAVIOR DELTA vs the fat base (inherent to snap machinery, flagged
not fixed): snapd's dbus activation entries hardcode User=root, so
the daemon runs as confined root instead of the fat base's geoclue
user (uid 203, still baked in the variant). All access is
interface-mediated either way. Note the geoclue polkit rules.d JS
file (geoclue->ModemManager allow) from the fat base is NOT
reproduced: ModemManager is absent on this image; if it ever lands,
that grant needs a polkit-side pass.

Consumers: gnome-shell (location indicator + the geoclue agent) and
gsd-datetime (timezone lookup) hold the session snap's `geoclue`
plug, connected to this snap's slot via gadget static connections.
