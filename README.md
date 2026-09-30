# Home SOC Lab

This is a small SOC environment I built at home to actually understand how log collection and detection work, instead of just reading about it. I'm a Cyber Security student working toward a SOC analyst role, and I wanted something hands-on that wasn't just a checklist tutorial.

## What's in it

I set up three virtual machines in VirtualBox: a Kali Linux box to act as the attacker, a Windows 10 machine as the victim, and an Ubuntu Server running Splunk Enterprise as the central SIEM. They're all connected on the same NAT network so they can talk to each other and I can access everything from my host machine.

The idea was simple on paper — get Windows logs flowing into Splunk, then simulate an attack and see if it shows up. In practice it took a lot longer than expected, mostly because of small configuration issues that don't show up until you actually try to use the thing.

## The problems I ran into (and how I fixed them)

The Splunk install itself broke the first time — a corrupted package left the system in a state where the service was "installed" according to the package manager but none of the actual files existed. I had to remove it cleanly and reinstall from the tarball instead of the .deb package.

Getting networking right took a bit of trial and error too. I originally set up port forwarding assuming a plain NAT adapter, but the VM was actually on a NAT Network, which handles port forwarding differently — the rules have to go through the NAT Network settings, not the adapter's own settings, and you need to specify a Guest IP.

The trickiest issue was the forwarder. I installed the Splunk Universal Forwarder on the Windows 10 machine, and it connected to Splunk just fine — I could see "Connected to idx=10.0.2.4:9997" in its logs. But nothing was showing up in Splunk. It took a while to realize the forwarder was connected but had never actually been told what to collect. The installer is supposed to create a config file (inputs.conf) based on which event logs you select during setup, but it never created one. I ended up writing it manually:

\[WinEventLog:Security]
disabled = 0
index = main

\[WinEventLog:Application]
disabled = 0
index = main

\[WinEventLog:System]
disabled = 0
index = main

Once that was in place and the forwarder restarted, logs started flowing in.

## Checking it actually works

I didn't want to just assume it was working, so I tested it two ways. First, I used Windows' built-in eventcreate command to generate a test log entry and watched it show up in Splunk within a minute. Then I tried something closer to a real scenario — deliberately failing a few logins on the Windows machine and searching for them:

index=main EventCode=4625
| stats count by Account\_Name, host

Event ID 4625 is a failed logon in Windows, and seeing it land correctly in Splunk was the first real confirmation that the whole pipeline — from event happening on the endpoint, to the forwarder picking it up, to Splunk indexing it — was actually working end to end.

## What's next

I want to add Sysmon to the Windows machine for deeper visibility into process creation and network connections, since Windows' default event logs don't capture that level of detail. I also want to get a forwarder running on the Kali box so I can pull in authentication logs from the Linux side and start correlating activity across both machines.

## A note on this being a lab

Everything here runs in isolated virtual machines on my own computer. Nothing touches a real network, and this is purely for learning.

