# Turning a Raspberry Pi Zero W into a Remote Gateway to My Home Network

I wanted a very specific kind of remote access for my home lab:

> When I am away from home, I want to reach devices on my home LAN
> without installing anything on those devices.

The obvious problem was that my home network uses `192.168.0.x`. Plenty
of networks I might connect to while travelling can use the same range.
If I simply advertised `192.168.0.0/22` through Tailscale, a laptop
connected to another `192.168.0.x` network could route that traffic
locally instead of sending it through my home VPN.

The solution I built was to use a tiny **Raspberry Pi Zero W** as a
Tailscale subnet gateway and present my home network remotely as a
different address space:

``` text
Remote device
      |
      | Tailscale
      v
192.168.10.0/24
      |
      v
Raspberry Pi Zero W
      |
      | NETMAP
      v
192.168.0.0/24
      |
      +---- Home devices
```

The important part is that the devices inside my home do not need
Tailscale installed. Only the Pi and the remote device need it.

------------------------------------------------------------------------

## What I wanted

The requirements were deliberately narrow:

- A Raspberry Pi Zero W already connected to my home Wi-Fi.
- No GUI.
- Minimal resource usage.
- Tailscale for the remote connection.
- No software installation on home PCs, cameras, NAS devices, Home
  Assistant, printers, etc.
- No port forwarding on the home router.
- Avoid the common `192.168.0.0/24` collision when travelling.
- Access home devices by IP as though they were on a separate remote
  subnet.

My actual home network is:

``` text
Home LAN:     192.168.0.0/22
Gateway:      192.168.0.1
Pi:           192.168.0.56
Interface:    wlan0
```

The Pi reported this routing table:

``` text
default via 192.168.0.1 dev wlan0 proto dhcp src 192.168.0.56 metric 600
192.168.0.0/22 dev wlan0 proto kernel scope link src 192.168.0.56 metric 600
```

The remote representation I chose is:

``` text
192.168.10.0/24
```

So, for example:

``` text
192.168.10.20  ->  192.168.0.20
192.168.10.30  ->  192.168.0.30
192.168.10.50  ->  192.168.0.50
```

------------------------------------------------------------------------

# 1. Keep the Pi lightweight

The Pi Zero W has no reason to run a desktop environment for this job.

I installed a headless Raspberry Pi OS Lite image and treated the Pi as
a network appliance rather than a general-purpose server.

The intended software footprint is basically:

``` text
Raspberry Pi OS Lite
        |
        +-- SSH
        |
        +-- Tailscale
        |
        +-- Linux IP forwarding
        |
        +-- iptables/NAT
```

No desktop, Docker stack, database, or application services are
required.

------------------------------------------------------------------------

# 2. Install Tailscale

Once the Pi was running and accessible over SSH:

``` bash
curl -fsSL https://tailscale.com/install.sh | sh
```

I then brought Tailscale up:

``` bash
sudo tailscale up
```

Tailscale displayed an authentication URL. After authenticating, I
verified the connection:

``` bash
tailscale status
```

The Pi appeared with its Tailscale address:

``` text
100.95.209.85  pi-gateway  ssuvamm@  linux  -
```

At this point the Pi was simply another device on my Tailscale network.

------------------------------------------------------------------------

# 3. Enable IP forwarding

The Pi needs to forward packets between Tailscale and the physical home
LAN.

I enabled IPv4 forwarding:

``` bash
sudo sysctl -w net.ipv4.ip_forward=1
```

Then made it persistent:

``` bash
echo 'net.ipv4.ip_forward = 1' | sudo tee /etc/sysctl.d/99-tailscale.conf
```

I also enabled IPv6 forwarding because it was part of the original
Tailscale forwarding configuration:

``` bash
echo 'net.ipv6.conf.all.forwarding = 1' | sudo tee -a /etc/sysctl.d/99-tailscale.conf
```

Then loaded the settings:

``` bash
sudo sysctl -p /etc/sysctl.d/99-tailscale.conf
```

Verification:

``` bash
sysctl net.ipv4.ip_forward
```

Result:

``` text
net.ipv4.ip_forward = 1
```

------------------------------------------------------------------------

# 4. Advertise a different subnet

This is the important part.

I did **not** advertise my actual home subnet:

``` text
192.168.0.0/22
```

Instead, I advertised:

``` text
192.168.10.0/24
```

with:

``` bash
sudo tailscale set --advertise-routes=192.168.10.0/24
```

I verified the setting:

``` bash
sudo tailscale debug prefs | sed -n '/"AdvertiseRoutes"/,+3p'
```

which showed:

``` text
"AdvertiseRoutes": [
        "192.168.10.0/24"
],
```

I then approved the advertised subnet route for `pi-gateway` in the
Tailscale admin console.

------------------------------------------------------------------------

# 5. Make the Pi treat 192.168.10.0/24 as its virtual LAN

Initially, Linux had no local route for the virtual subnet.

A lookup showed:

``` bash
ip route get 192.168.10.20
```

and initially returned:

``` text
192.168.10.20 via 192.168.0.1 dev wlan0 src 192.168.0.56
```

That was wrong for this design.

The Pi should treat `192.168.10.0/24` as the remote-side network
attached to Tailscale.

I added:

``` bash
sudo ip route add 192.168.10.0/24 dev tailscale0
```

Verification:

``` bash
ip route get 192.168.10.20
```

gave:

``` text
192.168.10.20 dev tailscale0 src 100.95.209.85
```

After reboot, this manually-added route was no longer visible in the
normal Linux routing table:

``` bash
ip route show 192.168.10.0/24
```

returned nothing.

However, remote access continued to work because Tailscale was handling
the advertised subnet route itself. This was confirmed by successfully
reaching home devices remotely.

------------------------------------------------------------------------

# 6. Translate 192.168.10.x to 192.168.0.x

Now the Pi needed to translate the remote address space to the actual
home address space.

The Raspberry Pi was using:

``` text
iptables v1.8.9 (nf_tables)
```

and the `NETMAP` kernel module was available:

``` bash
sudo modprobe xt_NETMAP
```

I then added a destination NETMAP rule:

``` bash
sudo iptables -t nat -A PREROUTING \
  -i tailscale0 \
  -d 192.168.10.0/24 \
  -j NETMAP \
  --to 192.168.0.0/24
```

This effectively creates:

``` text
192.168.10.x -> 192.168.0.x
```

For the reverse direction I added:

``` bash
sudo iptables -t nat -A POSTROUTING \
  -o tailscale0 \
  -s 192.168.0.0/24 \
  -j NETMAP \
  --to 192.168.10.0/24
```

The resulting rules were:

``` text
-A PREROUTING -d 192.168.10.0/24 -i tailscale0 -j NETMAP --to 192.168.0.0/24
-A POSTROUTING -s 192.168.0.0/24 -o tailscale0 -j NETMAP --to 192.168.10.0/24
```

------------------------------------------------------------------------

# 7. Verify the translation

I checked the NAT counters:

``` bash
sudo iptables -t nat -L -n -v
```

The PREROUTING rule had traffic:

``` text
48  2632  NETMAP  ...  tailscale0  ...  192.168.10.0/24
```

That was the first useful confirmation that traffic arriving through
Tailscale was actually hitting the translation rule.

------------------------------------------------------------------------

# 8. Make the firewall rules persistent

The NAT configuration should not disappear after a reboot.

I installed the persistence package:

``` bash
sudo apt update
sudo apt install -y iptables-persistent
```

Then explicitly saved the rules:

``` bash
sudo netfilter-persistent save
```

I verified that the rules were present:

``` bash
sudo iptables-save | grep -E 'NETMAP|192.168.10'
```

Result:

``` text
-A PREROUTING -d 192.168.10.0/24 -i tailscale0 -j NETMAP --to 192.168.0.0/24
-A POSTROUTING -s 192.168.0.0/24 -o tailscale0 -j NETMAP --to 192.168.10.0/24
```

------------------------------------------------------------------------

# 9. Make sure Tailscale starts at boot

I enabled the Tailscale daemon:

``` bash
sudo systemctl enable --now tailscaled
```

Then verified:

``` bash
systemctl is-enabled tailscaled
```

Result:

``` text
enabled
```

------------------------------------------------------------------------

# 10. Reboot test

I rebooted the Pi:

``` bash
sudo reboot
```

After reconnecting over SSH, I checked the system again.

The normal Linux routing table did not show a manually-added:

``` text
192.168.10.0/24 dev tailscale0
```

route, but remote access was still working.

That was the important test: **the system continued to work after reboot
without manually recreating the route.**

------------------------------------------------------------------------

# 11. Final verification

Tailscale:

``` bash
tailscale status
```

showed the Pi online:

``` text
100.95.209.85   pi-gateway      ssuvamm@  linux    -
```

The advertised route was still configured:

``` bash
sudo tailscale debug prefs | sed -n '/"AdvertiseRoutes"/,+3p'
```

Result:

``` text
"AdvertiseRoutes": [
        "192.168.10.0/24"
],
```

IPv4 forwarding was enabled:

``` bash
sysctl net.ipv4.ip_forward
```

Result:

``` text
net.ipv4.ip_forward = 1
```

And the NAT rules survived the reboot:

``` bash
sudo iptables-save | grep -E 'NETMAP|192.168.10'
```

Result:

``` text
-A PREROUTING -d 192.168.10.0/24 -i tailscale0 -j NETMAP --to 192.168.0.0/24
-A POSTROUTING -s 192.168.0.0/24 -o tailscale0 -j NETMAP --to 192.168.10.0/24
```

Finally, from a remote Tailscale device, I successfully:

``` text
ping 192.168.10.1
```

and accessed home devices through their translated `192.168.10.x`
addresses.

That confirmed the complete path was working.

------------------------------------------------------------------------

# The final architecture

``` text
                         INTERNET
                             |
                             |
                       Tailscale VPN
                             |
                +------------+------------+
                |                         |
         Remote laptop              Remote phone
         Tailscale client           Tailscale client
                |                         |
                +------------+------------+
                             |
                    192.168.10.0/24
                             |
                     +-------v-------+
                     | Raspberry Pi  |
                     |    Zero W     |
                     |               |
                     |  tailscale0   |
                     |  100.95.209.85|
                     |               |
                     |     NETMAP    |
                     +-------+-------+
                             |
                      wlan0 / Wi-Fi
                             |
                    192.168.0.0/22
                             |
              +--------------+--------------+
              |              |              |
         Home Assistant     NAS          Cameras
          192.168.0.x    192.168.0.x    192.168.0.x
```

The practical result is:

``` text
Remote:
192.168.10.20
      |
      v
Home:
192.168.0.20
```

So when I’m on another network that happens to use:

``` text
192.168.0.0/24
```

there is no address-space collision from my perspective. I use:

``` text
192.168.10.x
```

for my home devices.

------------------------------------------------------------------------

# What this gives me

The nice part of this setup is what I **didn’t** have to do.

I didn’t have to:

- install Tailscale on every home device
- configure VPN software on my NAS
- configure VPN software on Home Assistant
- expose services through port forwarding
- configure DDNS
- expose individual services to the public internet
- change the IP addresses of my existing home devices

The Pi Zero W becomes a small dedicated gateway:

``` text
Tailscale
    +
Linux forwarding
    +
NETMAP
    =
remote access to the home LAN
```

And because the Pi only has to handle networking, it doesn’t need much
CPU or RAM.

------------------------------------------------------------------------

# Important limitation: my home LAN is actually /22

There is one detail worth documenting carefully.

My home network is:

``` text
192.168.0.0/22
```

which covers:

``` text
192.168.0.0 - 192.168.3.255
```

The translation I implemented is:

``` text
192.168.10.0/24
        |
        v
192.168.0.0/24
```

So this setup gives a complete 1:1 mapping for the `192.168.0.x`
portion, but **not every address in the full `/22`**.

If I eventually need remote access to devices in:

``` text
192.168.1.x
192.168.2.x
192.168.3.x
```

I need to extend the design rather than assuming that the current `/24`
mapping covers the entire `/22`.

For the devices I tested, the current configuration was sufficient.

------------------------------------------------------------------------

# Final configuration checklist

``` text
[✓] Raspberry Pi Zero W
[✓] Headless Raspberry Pi OS Lite
[✓] Wi-Fi connected
[✓] SSH available
[✓] Tailscale installed
[✓] Tailscale authenticated
[✓] Tailscale starts at boot
[✓] IPv4 forwarding enabled
[✓] 192.168.10.0/24 advertised by Tailscale
[✓] Subnet route approved in Tailscale
[✓] NETMAP translation configured
[✓] iptables rules persisted
[✓] Pi rebooted successfully
[✓] Remote ping tested
[✓] Remote access to home device tested
```

## The one-line summary

> **I turned a Raspberry Pi Zero W into a Tailscale subnet gateway that
> exposes my `192.168.0.x` home network remotely as `192.168.10.x`,
> avoiding the common problem of overlapping private LAN ranges while
> requiring no VPN software on the devices inside my home.**
