# Cisco 2921 Router Configuration Recovery

## What Happened

I was working on my Cisco 2921 router when I noticed that my previous configurations were no longer showing up. The router was asking me to go through the initial setup process again, which was confusing because I had already saved my configuration.

I had configured DHCP, SSH, usernames, and other settings, so I wanted to figure out what happened before starting everything over.

## Checking My Saved Configuration

The first thing I checked was whether my configuration was still saved on the router.

```cisco
show startup-config
```
<img width="569" height="472" alt="Configuration Register TROUBLESHOOT" src="https://github.com/user-attachments/assets/c19d4c32-5c73-45d1-a358-353df4252d9b" />

The command displayed my previous settings, including my hostname, DHCP pool, DNS servers, and username.

This told me that the configuration hadn't actually been erased. It was still stored in NVRAM, but for some reason the router wasn't loading it.

## Checking the Router's Boot History

Next, I checked the router's system information.

```cisco
show version
```

Part of the output showed:

```text
Router uptime is 1 hour, 55 minutes
System returned to ROM by power-on
Last reload reason: power-on
```

I hadn't intentionally restarted the router, but the output showed that it had recently powered on.

At this point, I knew the router had restarted, but I still needed to understand why it wasn't loading my saved configuration.

## Finding the Problem

I checked the configuration register using:

```cisco
show version | include register
```

The output was:

```text
Configuration register is 0x2142
```

This was the problem.

The configuration register controls certain parts of the router's boot process. With the value set to `0x2142`, the router skips the startup configuration instead of loading it normally.

That explained why the router was acting like it had never been configured, even though my settings were still saved.

I wasn't able to determine what originally changed the register to `0x2142`.

## Fixing the Configuration Register

To correct the setting, I entered global configuration mode and changed the register to `0x2102`.

```cisco
configure terminal
config-register 0x2102
end
```

I then checked the register again.

```cisco
show version | include register
```

The router returned:

```text
Configuration register is 0x2142 (will be 0x2102 at next reload)
```

This showed that the new value had been accepted and would take effect after the next restart.

I also avoided saving the current running configuration over the startup configuration because I didn't want to overwrite the settings I was trying to recover.

## Current Status

I found the cause of the problem and corrected the configuration register.

The remaining step is to restart the router and confirm that it loads the original configuration normally.

Until that test is completed, I'm treating the recovery as partially verified.

## What I Learned

The biggest thing I learned from this issue is that a router can appear to have lost its configuration without actually losing it.

Before rebuilding everything, it's worth checking both the running configuration and the startup configuration.

I also learned how the configuration register affects the boot process and why `0x2142` and `0x2102` behave differently.

This was a useful troubleshooting experience because the problem wasn't with DHCP, SSH, or any of the individual configurations I had been working on. It was related to how the router started up and whether it loaded those configurations in the first place.
