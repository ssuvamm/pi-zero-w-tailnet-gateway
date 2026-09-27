# Raspberry Pi Zero W as a Tailscale Home Network Gateway

I wanted to access my home network remotely without installing Tailscale on every device inside it. A Raspberry Pi Zero W turned out to be enough.

But there was a second problem I didn't initially account for: **IP range conflicts** when connecting from another network.

## 1. The Problem

### Access devices that can't run Tailscale

Some devices on a home network don't need—or can't easily run—a Tailscale client:

* Home Assistant
* NAS
* IP cameras
* printers
* IoT devices
* random network appliances

Tailscale has a solution for this: a **subnet router**. One device runs Tailscale and forwards traffic to the rest of the LAN.

The Pi Zero W is perfect for this because it only needs to handle routing.

My home network was:

```text
192.168.0.0/22
```

and the Pi was:

```text
192.168.0.56
```

I enabled IP forwarding:

```bash
sudo sysctl -w net.ipv4.ip_forward=1
```

and made it persistent:

```bash
echo 'net.ipv4.ip_forward = 1' | sudo tee /etc/sysctl.d/99-tailscale.conf
sudo sysctl -p /etc/sysctl.d/99-tailscale.conf
```

Then installed and authenticated Tailscale:

```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up
```

Finally, I advertised the home subnet:

```bash
sudo tailscale set --advertise-routes=192.168.0.0/22
```

That solves the first problem.

But it introduced another one.

## 2. The IP Range Problem

Imagine I'm travelling and connect to a hotel Wi-Fi using:

```text
192.168.0.0/24
```

That's also where my home network lives.

Now if I try to access:

```text
192.168.0.50
```

my laptop may think:

> "That's on the network I'm currently connected to."

It doesn't necessarily send the traffic through Tailscale.

I needed my home network to have a **different address space from the perspective of the remote device**.

So I created a virtual remote representation:

```text
Remote view              Actual home network

192.168.10.20    ────►   192.168.0.20
192.168.10.30    ────►   192.168.0.30
192.168.10.50    ────►   192.168.0.50
```

I advertised:

```bash
sudo tailscale set --advertise-routes=192.168.10.0/24
```

and used Linux `NETMAP` to translate the addresses.

```bash
sudo modprobe xt_NETMAP
```

```bash
sudo iptables -t nat -A PREROUTING \
  -i tailscale0 \
  -d 192.168.10.0/24 \
  -j NETMAP \
  --to 192.168.0.0/24
```

with the reverse translation:

```bash
sudo iptables -t nat -A POSTROUTING \
  -o tailscale0 \
  -s 192.168.0.0/24 \
  -j NETMAP \
  --to 192.168.10.0/24
```

I persisted the rules with:

```bash
sudo apt install -y iptables-persistent
sudo netfilter-persistent save
```

Now remotely I can use:

```text
192.168.10.x
```

while the actual devices at home remain:

```text
192.168.0.x
```

No changes are required on those devices.

## 3. Why I Cared — And You Might Too

The subnet-router part is useful on its own. The address translation is the part that makes this setup much more practical.

Private IP ranges are reused everywhere.

Your home might be:

```text
192.168.0.x
```

while a hotel, office, friend's house, or another lab uses exactly the same range.

A VPN that simply routes `192.168.0.x` can run straight into that collision.

Giving the home network a separate **remote-only address space** avoids the ambiguity:

```text
Local network I'm currently on
        ↓
    192.168.0.x

My home, accessed remotely
        ↓
   192.168.10.x
```

And because the translation happens on the Pi, the devices at home don't need to know anything about it.

## 4. Conclusion

The final setup is deliberately simple:

```text
Remote device
      │
      │ Tailscale
      ▼
192.168.10.0/24
      │
      ▼
Raspberry Pi Zero W
      │
      │ NETMAP
      ▼
192.168.0.0/24
      │
      ├── NAS
      ├── Home Assistant
      ├── Cameras
      ├── Printers
      └── IoT
```

The Pi Zero W doesn't need a desktop, and the devices inside the house don't need Tailscale clients.

> **A $15-ish Raspberry Pi Zero W can act as a tiny Tailscale gateway for an entire home network—and with address translation, it can make that network reachable even when the network you're visiting uses the same IP range.**
