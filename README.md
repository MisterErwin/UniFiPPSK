# UniFiPPSK

UniFi does support RADIUS mac authentication for WPA2 personal,
but does not support individual PSKs (out of the box).
Other vendors offer this,
sometimes marketed as Dynamic Pre-shared Key (DPSK), Identity PSK (IPSK), or Private PSK (PPSK).

[This thread](https://community.ui.com/questions/Proof-of-Concept-Private-PSK-Personal-PSK-PPSK-with-dynamic-VLAN-via-RADIUS-MAC-auth/68f3097a-dcc1-4c31-bb51-ede39e706e30)
on the UniFi forums outlines a POC for Private PSKs with dynamic VLAN assignment.

Since this POC has been published, Ubiquiti has introduced [UID](https://ui.com/uid) which comes with its
own [kind of WiFi access](https://help.ui.com/hc/en-us/articles/4405534998295-UID-Manage-UID-WiFi) - one where the PSK
is unique per user.

Due to the addition of UID,
the controller will provision the required *system.cfg* lines itself,
if the network is marked as a UID IoT network.
This is usually only possible via the UDM, but by modifying the mongoDB directly,
it is possible to enable the UID IoT mode:

In case you already have set up a wireless network with RADIUS mac authentication,
you can skip ahead.
Create a new WPA Personal wireless network using the UniFi web UI, enter a name,
and an arbitrary passphrase (it will not be used, but the controller requires one set.).

In the radius mac authentication section select a RADIUS profile.
(If required, enable RADIUS assigned VLANs in the profile).

To morph your plain WPA 2 personal network with a fixed passphrase to one using PPSK,
we have to connect to the MongoDB used by the controller.

`mongo --port 27117`

First, we set the `attr_hidden_id` to `UidIot`,
then enable the option to retrieve the WPA passphrase from RADIUS,
and finally require the dynamic VLANs (only required for RADIUS assigned VLANs).

```js
use
ace
db.wlanconf.update(
    {name: "Your Wireless Network Name"},
    {
        $set: {
            "attr_hidden_id": "UidIot",
            "wpa_psk_radius": "required",
            "vlan_wlan_mode": "required"
        }
    }
)
```

Your radius' users file should look like the following and include the `Tunnel-Password` option.

```
aa:bb:cc:dd:ee:ff Cleartext-Password := "aa:bb:cc:dd:ee:ff"
    Tunnel-Type = 13,
    Tunnel-Medium-Type = 6,
    Tunnel-Private-Group-Id = 1234,
    Tunnel-Password = ILikeTrams
```

Now you just have to (force-)provision all APs for this change to take effect.

Make sure to use an updated firmware (should be a v6 firmware) and controller,
as otherwise you will be greeted by "UID IoT WLAN Your Wireless Network Name is not supported by f4:92:aa:bb:cc:dd and
will be skipped" errors in your server.log.

