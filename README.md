# TOR Connection Check for Nagios/Icinga

## What is it

This is a plugin for Nagios / Icinga to monitor the connection of your TOR daemon to the TOR network.

## What it does

The plugin tries to fetch a web page over the TOR network via the SOCKS interface. The default queried site is `https://www.torproject.org`. Basically a HTTP HEAD request is sent. This is enough because the returned content is irrelevant. I do this to take away unnecessary network load from the TOR network.

## How to install
### Step 1: Get the script

Fetch the script and place it in your PluginDir-Folder. That's usually `/usr/lib/nagios/plugins`

### Step 2: Install dependencies

I assume `perl` is already installed.

Furthermore we need two more perl libraries:
- LWP::UserAgent
- LWP::Protocol::socks

If you use `apt` you can simply install it via

```apt-get install liblwp-protocol-socks-perl```

Another (package manager independant) way is via CPAN. Your choice!

### Step 3: Configure TOR daemon

To allow fetching a site via TOR you have to allow SOCKS requests to the TOR daemon.
The settings depend on your setup.

#### Setup 1: Running the script on the same system as TOR

In a perfect environment your TOR client is not listening for SOCKS connections. Well, we need this feature. In this scenario the
script is on the same server as the TOR daemon. The most secure way is listening only on 127.0.0.1:9050 and only allow 127.0.0.1
to send SOCKS requests.

Your tor config file should look like this:
```
[...]
SocksPort 127.0.0.1:9050

[...]
# Allow only localhost to connect
SocksPolicy accept 127.0.0.1/32
SocksPolicy reject *
```

The check has then to be done via by_ssh as check command or through an icinga daemon.

#### Setup 2: Running the script on another system

I have this setup at home. For my LAN I've defined the address range 192.168.0.0/24.

TOR-Server has the IP 192.168.0.250

Icinga-Server has the IP: 192.168.0.247

The `check_tor`-File is only on the icinga server but has to connect to the TOR-server.

My tor config file looks like this:
```
[...]
SocksPort 127.0.0.1:9050
SocksPort 192.168.0.250:9050

[...]
# Allow only localhost to connect
SocksPolicy accept 127.0.0.1/32
SocksPolicy accept 192.168.0.0/24
SocksPolicy reject *
```

I'm allowing every client on my LAN to make SOCKS requests to the TOR daemon.

In this case my icinga daemon can execute the check locally.

### Step 4: Make it available for Icinga 2

#### Step 4.1: Create a CheckCommand object

Navigate on your Icinga 2 server to your config folder, e.g.: `/etc/icinga2/conf.d`.

Place this block of code into it:

```
object CheckCommand "tor" {
        import "plugin-check-command"

        command = [ PluginDir + "/check_tor" ]

        arguments = {
                "--host" = "$tor_host$"
                "--port" = "$tor_port$"
                "--target" = "$tor_target$"
        }
}
```

Adjust `command` according to the path in your system if you didn't place the script in `/usr/lib/nagios/plugins`!

#### Step 4.2: Add check to a host

To add the check to a host use this snippet in a host configuration file:

```
object Host "proxy" {
	[...]
}

object Service "tor" {
        host_name = "proxy"
        display_name = "TOR connection"

        check_command = "tor"
        check_interval = 1h

        vars.tor_host = "proxy.veloc1ty.lan"
}
```

Possible values are:

* tor_host = The address (hostname or IP) of the TOR daemon (optional, default: 127.0.0.1)
* tor_port = The port where the TOR daemon listens (optional, default: 9050)
* tor_target = The target site (optional, default: https://www.torproject.org)

See Step 3 to check if you have to set tor_host or tor_port depending on your setup.

If you want to add it to multiple hosts work with `apply`!

## Check connection based on a hidden service

The check is fetching the TOR project website from the "normal" web by default.

You can also define a hidden service (.onion) if you want. Just set the `tor_target` variable.
With this option you can also monitor the reachability of a hidden service.

## Performance data

This check provides the latency in milliseconds (fetch delta time) as performance data.