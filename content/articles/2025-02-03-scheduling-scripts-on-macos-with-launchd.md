+++
title = "Scheduling Scripts on macOS with launchd"
summary = "In which I detail the process of automating backups of the Apple Photos library on macOS using launchd."
date = "2025-02-03"
tags = ["apple-photos", "backup", "launchd", "macos"]
+++

While I've used Google Photos extensively as a central storage for my personal photos, Apple Photos has recently taken over this responsibility. This article explains how I replaced an always-on service that archived my Google Photos with a minimally viable macOS launchd agent for Apple Photos. 

It's great that we can rely on Google and Apple to keep our photos as a one copy, one storage medium, and one off-site location, but that's not enough to meet the [“3–2–1 rule”](https://library.vassar.edu/specialcollections/recordsmanagement#:~:text=This%203%2D2%2D1%20backup,hard%20drive%2C%20or%20USB%20drive.). We also need local backups that we’re continuously synchronizing to act as a second copy and a second storage medium.

Apple Photos exists within the confines of iCloud and, as a consequence, if you want to automate backups in the same way that you can with GPhotos, you need to provide projects like [this](https://github.com/mandarons/icloud-docker) or [this](https://github.com/steilerDev/icloud-photos-sync) your iCloud password. With Jake Wharton's `docker-gphotos-sync` [project](https://github.com/JakeWharton/docker-gphotos-sync), you are free to use your real password, or (preferably) create an app-specific password, sandboxing potential incidents from spreading further. There does not seem to exist a similar tool for Apple Photos. Although I would prefer a more versatile and reliable tool, hacking something fast was more important.

Since I have the entirety of my library local, I could define a job on my [crontab](https://en.wikipedia.org/wiki/Cron) to clone its contents to another medium using standard CLI tools. However, cron is no longer the recommended approach for running headless tasks on a Mac. Per Apple's [documentation](https://developer.apple.com/library/archive/documentation/MacOSX/Conceptual/BPSystemStartup/Chapters/CreatingLaunchdJobs.html):

>If you are running per-user background processes for OS X, launchd is also the preferred way to start these processes. These per-user processes are referred to as user agents. A user agent is essentially identical to a daemon, but is specific to a given logged-in user and executes only while that user is logged in.

The scheduling method used in macOS is not only preferable, but also significantly better than traditional approaches such as cron jobs. For example, if a computer is asleep when a task is scheduled to run, macOS will automatically reschedule the job to execute once the system wakes up. In contrast, cron jobs are completely ignored and only run when the computer is awake.

It's not particularly complicated to set up a _hello world_ for a user's launch agent ([example](https://wiki.freepascal.org/macOS_daemons_and_agents#Launch_Daemon_to_run_a_script_as_a_user_on_a_schedule)), so I started with a simple shell script that invoked [rsync](https://rsync.samba.org) to mirror my Photos Library directory to my NAS. But recent security improvements in macOS have tightened the system up to a point where much of the internet literature became outdated. An example is [not being able to access](https://apple.stackexchange.com/questions/338213/how-to-run-a-launchagent-that-runs-a-script-which-causes-failures-because-of-sys) user's directories (such as `~/Pictures` or `~/Documents`) via terminal emulators unless they've been given the _Full Disk Access_ permission. The specific error I got was `rsync: opendir "~/Pictures/Photo Library" failed: Permission denied (13)`.

A workaround would be to disable [SIP](https://en.wikipedia.org/wiki/System_Integrity_Protection) or [allow bash/zsh FDA](https://support.apple.com/guide/mac-help/change-privacy-security-settings-on-mac-mchl211c911f/mac), but a much safer approach would be to allow only rsync access these directories. Unfortunately, even this did not solve the issue, as confirmed by several developers in the Stack Overflow thread linked above. A suggested workaround was to create a dedicated application that runs the script and give it FDA. This approach worked for me on macOS Sequoia 15.3.

In short, you want the following:

- An application (I used Automator.app to create one) that runs a shell script that executes rsync with the correct arguments.
- A .plist file inside `~/Library/LaunchAgents/` that invokes this application [^syntax].

[^syntax]: The command `plutil -lint <plist-file>` can be used to validate the syntax.

In my case, the application contained [^single-quotes]:

[^single-quotes]: The pair of single quotes above is necessary to preserve the space of the destination path, which is interpreted on the target machine.

```shell
#!/bin/sh
/usr/bin/rsync -azv --delete-after \
  "/Users/leandro/Pictures/Photos Library.photoslibrary/" \
  mynas:\''Photos Library.photoslibrary/'\'
```

With `local.PhotoSync.plist` at `~/Library/LaunchAgents/` being:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>local.PhotoSync</string>
    <key>ProgramArguments</key>
    <array>
        <string>/usr/bin/open</string>
        <string>/Users/leandro/Applications/PhotoSync.app</string>
    </array>
    <key>StartCalendarInterval</key>
    <dict>
        <key>Hour</key>
        <integer>0</integer>
        <key>Minute</key>
        <integer>0</integer>
    </dict>
</dict>
</plist>
```

to run at every midnight, as specified with `StartCalendarInterval`. With them in place, the commands to register[^register] & unregister the agent on `launchd` are:

[^register]: the `-k` argument kills any running instance before restarting the service.

```shell
launchctl load -k <your-plist>

launchctl unload <your-plist>
```

The man pages of [launchctl](https://ss64.com/mac/launchctl.html) and [launchd.plist](https://manpagez.com/man/5/launchd.plist/) are fairly easy to understand and cover a lot of details not mentioned in this article.

By setting an automated, in-background process that constantly replicates my photos to a secure offsite storage, I've removed Apple as a single point of failure in the event of unexpected account closure.

I believe that the security and preservation of our digital memories should be a top priority and were we must actively more actions than to rely on a free service to magically exist and serve us.
