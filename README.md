# Windows-Server-Hardening-Lab
Hands-on Windows Server hardening lab — removing an unused virtual device, uninstalling unnecessary software, and pulling the FTP role out of IIS, plus fixing a broken hostname redirect on Kali via /etc/hosts.

Windows Server Hardening Lab (Device, Software & Role Removal) + Hosts-File Redirection on Kali

Windows Server | Device Manager | Server Manager | IIS / FTP | Attack Surface Reduction | Kali Linux | /etc/hosts | wget

OVERVIEW
This one was about reducing attack surface on a Windows Server, which is one of those tasks that sounds almost too basic to write up, until you actually sit down and do it properly. The idea is simple: if a device, a program, or a service isn't needed, it shouldn't be sitting there, because every one of those things is a little more surface for someone to poke at. I went after all three angles on the same domain-joined server, MS10.ad.structureality.com - a virtual DVD-ROM nobody uses anymore, a diagnostic utility that had no business being installed on a server in the first place, and an FTP role service that was still enabled even though nothing needed it. On the side, I also spent some time on a Kali box working out why a hostname wasn't resolving to the right place, which turned into a decent little lesson in reading error messages properly instead of just re-running the same command and hoping.

I screenshotted each confirmation dialog and result screen as I went. Partly for this write-up, but mostly because it's too easy to click through a warning dialog on autopilot and not actually register what it said.

OBJECTIVE
Strip an unnecessary virtual device, an unneeded application, and an unused server role off a Windows Server, paying attention to what each removal warning actually says before confirming it. Then, separately, use a Linux client to show how a wrong entry in /etc/hosts breaks reachability, and how fixing it looks different from a DNS problem.

ENVIRONMENT
- Lab platform: CompTIA Labs (Lab 21 - APPLIED - Hardening), run through the CompTIA Learning Platform
- Windows target: MS10 (MS10.ad.structureality.com), domain-joined
- Linux client: Kali Linux, root shell
- Target web app: JuiceShop, at juiceshop.local
- Date: 23 July 2026

Hostnames and IP ranges here are whatever the lab environment happened to spin up for this session, so don't expect them to mean anything outside of it.

TOOLS I USED
- Device Manager - for finding and getting rid of the unused virtual DVD-ROM
- Control Panel > Programs and Features - for uninstalling CPU-Z, which had no reason to be on a server
- Server Manager's Remove Roles and Features Wizard - for pulling the FTP Server role service back out of IIS
- Kali's terminal, plus nano and wget - for fixing and then testing the /etc/hosts entry

WHAT I DID

Quick summary before I get into the weeds:

Component                    | Where                                 | What I did                            | What Windows asked me
------------------------------------------------------------------------------------------------------------------------------------------------
Microsoft Virtual DVD-ROM    | Device Manager > DVD/CD-ROM drives    | Disabled it, then uninstalled it      | "Disabling this device will cause it to stop functioning..." then "Confirm Device Uninstall"
CPUID CPU-Z 2.06             | Programs and Features                 | Uninstalled it                        | "Are you sure you want to completely remove CPUID CPU-Z and all of its components?"
FTP Server role service      | Server Manager > Web Server (IIS)     | Removed via the wizard                | "...this server restarts automatically, without additional notifications. Do you want to allow automatic restarts?"

Device Manager: the virtual DVD-ROM nobody needed
First thing I did was check whether the driver was even out of date, mostly out of habit - right-click, Update Driver Software, search automatically. Windows came back with "the best driver software for your device is already installed," which told me the device itself wasn't the problem, it was just unnecessary. So I went to Disable next, and got the standard warning that disabling it would stop it functioning, did I really want to. Fair question, and worth pausing on for an actual server rather than clicking Yes on reflex. Since there was no reason to keep it around even in a disabled state, I went the rest of the way and uninstalled it, which threw one more confirmation ("you are about to uninstall this device from your system") before it actually came out.

Programs and Features: CPU-Z had to go
This one's pretty straightforward. CPU-Z is a CPU/hardware inspection tool - genuinely useful on your own machine, completely pointless on a server, and one more thing that has to be patched and accounted for if it stays. Uninstall, confirm "are you sure you want to completely remove CPUID CPU-Z and all of its components," done.

Server Manager: pulling FTP out of IIS
This one took a bit more care because I didn't want to rip out the whole IIS role, just the FTP piece. In Manage > Remove Roles and Features, I expanded Web Server (IIS) > FTP Server and unchecked just the FTP Service, leaving Web Server and Management Tools alone since the box still needed to serve web content, just not FTP. The confirmation page spelled out exactly what was leaving: Web Server (IIS) > FTP Server > FTP Service. Windows also asked whether it could restart automatically if needed without giving me another heads-up - I said yes, since it was a lab box and I'd rather it finish cleanly than sit half-removed waiting on a restart I forgot to do. Watched it chew through the removal progress bar until it reported done.

FTP's a decent example of why this kind of cleanup matters beyond just tidiness: it's plaintext by default, it tends to get left switched on long after anyone's actually using it, and it's a well-worn target for brute-force and credential sniffing. Pulling the role service out entirely beats just turning off the site and hoping nobody flips it back on.

Kali: chasing down a hostname that wouldn't resolve
Ran wget juiceshop.local expecting it to just work, and instead got "No route to host" after it resolved to 203.0.113.249. That's a useful message to actually read - it's not a DNS failure, the name resolved fine, it's that the box can't reach that particular address at all. So I opened /etc/hosts in nano, found the entry was pointing at the wrong IP, corrected it to 203.0.113.228, saved (hit Y at the "save modified buffer?" prompt), and ran wget again. This time it resolved to the corrected address, connected, came back with a clean 200 OK, and dropped index.html (1.9K) onto disk.

Worth saying plainly: that first attempt failed because I hadn't fixed the hosts file yet, not because of anything fancier. I'm leaving that in rather than skipping straight to the working version, because "No route to host" versus a DNS failure versus a connection refused are three different problems that look similar if you're not paying attention, and telling them apart is most of the actual skill here.

CHECKING IT ACTUALLY STUCK
- Ran Scan for hardware changes again in Device Manager - the virtual DVD-ROM didn't come back.
- Went back into Programs and Features - CPU-Z was gone from the list.
- Server Manager's removal progress page confirmed Web Server (IIS) > FTP Server > FTP Service had come out clean, no errors.
- On Kali, cat /etc/hosts to make sure the corrected line (203.0.113.228 juiceshop.local) had actually saved, then confirmed with another wget that it was still resolving and reachable.

WHAT'S IN THIS REPO
windows-server-hardening-lab/
  README.txt                                  - this file
  screenshots/
    01-device-manager-dvd-rom.png             - Microsoft Virtual DVD-ROM under DVD/CD-ROM drives
    02-update-driver-already-installed.png    - "best driver software is already installed"
    03-disable-device-warning.png             - disable confirmation dialog
    04-confirm-device-uninstall.png           - uninstall confirmation dialog
    05-scan-for-hardware-changes.png          - re-scan after removal
    06-cpuz-uninstall-confirm.png             - Programs and Features, CPU-Z uninstall prompt
    07-remove-roles-wizard-ftp.png            - Server Roles page, FTP Service unchecked
    08-confirm-removal-restart-prompt.png     - automatic restart prompt
    09-removal-progress-results.png           - feature removal completed
    10-wget-no-route-to-host.png              - failed wget with wrong IP
    11-nano-etc-hosts-edit.png                - correcting the hosts entry
    12-wget-200-ok.png                        - successful wget after the fix

WHAT I ACTUALLY LEARNED FROM THIS
Honestly, the biggest thing wasn't any single click - it was slowing down enough to read what each dialog was actually telling me instead of treating "Yes/OK/Confirm" as one button. A disable warning and an uninstall warning aren't interchangeable, and on a real server the difference matters. I also got a clearer sense of how Server Manager separates a role from a role service - being able to pull just FTP out from under Web Server (IIS) without touching the rest of it felt like a small thing but it's the kind of distinction that stops you from breaking something you didn't mean to. And on the networking side, "No route to host" finally clicked as its own category of failure, separate from DNS not resolving or a connection getting actively refused - which sounds obvious written down, but isn't always obvious at 6am staring at a terminal.

WHY THIS MATTERS OUTSIDE A LAB
Most of the interesting breaches don't start with something exotic - they start with something that was left switched on, installed, or reachable that nobody remembered was there. An FTP role nobody's used in a year, a diagnostic tool someone installed once and forgot about, a device that serves no purpose but is technically still attack surface - none of it's glamorous, all of it adds up. The /etc/hosts side of this maps onto real troubleshooting too: pointing a client at a specific box for testing, or figuring out that a "the server's down" complaint is actually just a stale entry on someone's machine.

WHERE I'M COMING FROM
I'm coming into cybersecurity from a healthcare background, which sounds like a strange jump on paper but a lot of the instincts carry over - following procedure properly, being careful with sensitive stuff, staying level-headed when something isn't behaving. I'm working through CompTIA Security+ at the moment and putting together labs like this one to build up the hands-on side that my CV doesn't show yet.

WHAT I WANT TO GET INTO NEXT
- Hardening against an actual published baseline (CIS Benchmarks) instead of eyeballing what looks unnecessary
- Doing this kind of removal with PowerShell instead of clicking through wizards by hand
- Group Policy for locking down device classes and removable media across more than one machine at a time
- AppLocker whitelisting, so it's not just about removing the obviously unnecessary stuff but only allowing what's explicitly approved
- Getting quicker at telling routing, firewall, and DNS failures apart without having to think about it

WHAT I'D DO DIFFERENTLY FOR REAL
- This was a guided lab - the device, the app, and the role were already sitting there waiting to be found. A real hardening pass starts with an inventory, not a list someone already handed you.
- I did all of this by hand, one thing at a time. In production I'd want it scripted so the same standard gets applied consistently, not just to whichever server I happened to click through that day.
- I said yes to the automatic restart without thinking about whether anyone else was using that box. On a real server that's its own incident waiting to happen - should've scheduled it properly instead.
- The hosts-file fix was a one-off manual check. I'd want that kind of reachability test logged somewhere, not just something I ran twice in a terminal and moved on from.

REFERENCES
- Microsoft Learn: Manage devices and drivers using Device Manager - https://learn.microsoft.com/windows-hardware/drivers/devtest/device-manager
- Microsoft Learn: Install or Uninstall Roles, Role Services, or Features - https://learn.microsoft.com/windows-server/administration/server-manager/install-or-uninstall-roles-role-services-or-features
- Microsoft Learn: FTP Server overview (IIS) - https://learn.microsoft.com/iis/ftp-server/
- GNU nano documentation - https://www.nano-editor.org/dist/latest/nano.html
- CompTIA Security+ (SY0-701) Exam Objectives - https://www.comptia.org/certifications/security
