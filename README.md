# TOR Connection Check for Nagios/Icinga

## What is it

This is a plugin for Nagios / Icinga to monitor the connection of your TOR deamon.

## What it does

This plugin tries to fetch a web page (or an .onion hidden service) over the TOR network.

## How it works

Via the `LWP` perl library a HTTP HEAD request is done over TOR's SOCKS interface. The default queried site is `https://www.torproject.org`. Of course you can set your own target. See section 4.2 for more info.

## How to install
### Step 1: Get the script

__Warning: This instructions only apply to Icinga 2__

Fetch the script and place it in your `PluginDir`-Folder. That's usually `/usr/lib/nagios/plugins`.
If you are not sure about the value of `PluginDir` you can check `/etc/icinga2/constants.conf`.

### Step 2: Install dependencies

I assume `perl` is already installed.

Furthermore we need two more perl libraries:
- LWP::UserAgent
- LWP::Protocol::socks

If you use `apt` you can simply install it via `apt-get install liblwp-protocol-socks-perl`

Another (package manager independant) way is via CPAN. Your choice!

### Step 3: Configure TOR daemon

Before someone can use the SOCKS interface of the TOR daemon you have to allow that explicitly!
The following setups are done in the TOR config file (Normally `/etc/tor/torrc`).

#### Setup 1: Running the script on the same system as TOR

In a perfect environment your TOR client is not listening for SOCKS connections. Well, we need this feature. In this scenario the
script is on the same server as the TOR daemon. The most secure way is listening only on 127.0.0.1:9050 and only allow 127.0.0.1/32
to connect to the SOCKS interface.

Your TOR config file should look like this:
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

In this case my icinga daemon can locally execute the check and set the host manually by a command argument.

### Step 4: Make it available for Icinga 2

#### Step 4.1: Create a CheckCommand object

Navigate on your Icinga server to your config folder, e.g.: `/etc/icinga2/conf.d` and open the `commands.conf` file.
Place this piece of code into it:

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

#### Step 4.2: Add check to a host

To add the check to a host use this snippet in a host configuration file:

```
object Host "proxy" {
	[...]
}

object Service "tor" {
        host_name = "proxy"
        display_name = "TOR status"

        check_command = "tor"
        check_interval = 1h
}
```

Possible variable values are:

* tor_host = The address (hostname or IP) of the TOR daemon (optional, default: 127.0.0.1)
* tor_port = The port where the TOR daemon listens (optional, default: 9050)
* tor_target = The target site (optional, default: https://www.torproject.org)

See step 3 to check if you have to set tor_host or tor_port depending on your setup.

If you want to add it to multiple hosts work with `apply`!

## Check connection based on a hidden service

You can "misuse" this plugin to check the reachability of an .onion hidden service, too.
I highly do not recommend this. TOR hidden service descriptions shared in the network have an expiry date.

In some circumstances this plugin will tell you that a hidden service is up even than it isn't. You would have to patch this script to rebuild all TOR circuits before the request is sent out to achieve "good" check results. You also have to reduce the check interval.
I'm currently developing another check plugin just for this case so please be patient. That's more complex than just doing a HTTP HEAD request.

## Performance data

This check provides the latency in milliseconds (fetch delta time) as performance data.
