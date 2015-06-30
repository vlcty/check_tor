# TOR Connection Check for Nagios/Icinga

## What is it

This is a plugin for Nagios / Icinga to monitor the connection to the TOR network of your TOR client.

## What it does

The plugin tries to fetch a web page over the TOR network via the SOCKS interface. The default queried site is `https://www.torproject.org`. I don't want to lie: The plugin just does a HTTP HEAD request to reduce the network load and we don't analyse it's content.

## How to install
### Step 1: Get the script

Fetch the script and place it in your PluginDir-Folder. Usually it's unter `/usr/lib/nagios/plugins`

### Step 2: Install dependencies

I assume you have `perl` already installed.

Furthermore we need two more perl libraries:
- LWP::UserAgent
- LWP::Protocol::socks

If you use `apt` you can simply install it via

```apt-get install liblwp-protocol-socks-perl```

Or you install it over CPAN. Your choice!
