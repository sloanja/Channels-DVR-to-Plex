# Channels-DVR-to-Plex

Channels DVR (https://community.getchannels.com/dvr/) is an extremely user friendly piece of software for recording TV from Silicondust HDHomeRun network TV tuners and, primarily, for serving to the Channels app on the Apple TV 4.  However, it is somewhat limited in its ability to serve to other clients and outside of a local network. I have found it convenient to automatically transcode recorded shows to an h.264 format and add them to a Plex (http://plex.tv) server.  This avoids the need for live transcoding to most devices, enabling lower power hardware and more optimized algorithms to minimize bandwidth.

Please note that this has not been thoroughly tested on all systems, and that it is to be used at your own risk. It has so far been tested on (at least) Ubuntu 16.04 Xenial (arm64) and Mac OS Sierra 10.12 (intel x86-64).  This is sub-beta quality right now!  Feel free to take it and make it your own, or contribute to this archive, as long as you share your work and operate within the license.

Ideally this should be installed using the *install.sh* script, thus:

curl -f -s https://raw.githubusercontent.com/karllmitchell/Channels-DVR-to-Plex/master/install.sh | bash

Note that this can also be used to update existing configurations.  However, if the script asks for a destination directory then your installation is probably sufficiently old or non-standard that you should do it manually if you want to preserve your settings.  Also, if you are a new user, consider the question about number of days backlog to transcode carefully, as this can take a long time. DAYS=0 is pretty safe, and you can always transcode missed shows later from the command line.

**The main script**

*channels-transcoder.sh* requires bash, and is designed primarily to run as a nightly job, preferably after all commercial scanning is over (12:01 AM by default), although it can be run from the command line too.  It can be installed either on the same machine as runs Channels DVR, or on another machine as long as you set the HOST variable; Note that it will copy files across the network in this latter mode.  The advantage of this is that you can run a very low powered machine (ARM board or NAS) for the recording, and then use another higher-powered machine for the transcoding, potentially letting it sleep most of the time. By default it lives in /usr/local/bin.

At the core of the script is HandBrakeCLI, which performs the transcoding via libx264. This is easy to obtain (http://handbrake.fr/ or via apt-get, macports, etc.) and by default I have it set up to produce high quality full resolution outputs that look good on a full HDTV with Apple TV, which also runs on most devices that are capable of 1080p playback.  Both subtitles and sound are preserved from the original MPEG, and if surround sound exists then a stereo track is added for more universal compatibility. 

Although the code will run on extremely underpowered systems, including low cost ARM-based SOCs runnings Linux, by default I do not recommend anything with less than 1 GByte, preferable 2 GBytes, of RAM (certainly at least 750 MBytes unused).  If you are accessing the inputs files across a network, you will want a fast one (Gigabit throughout, ideally), or to set aside at least a few tens of GBytes of storage and use the TEMP_COPY=1 argument.  Many modern intel systems can almost certainly process faster-than-realtime (i.e. a 1-hr show will take less than 1-hr), but an ARM SOC like a Raspberry Pi would probably be about 6x slower than real-time, and might not keep up with your TV viewing.

**Set up and first run**

The script must be run as a user with superuser access.  Do make sure that you can write to the destination directory and read from the source directory (if held locally).  It accepts various settings from a mandatory preferences file, which can also be overridden on the command line (see below).  If the installation script is not used, make sure to rename the channels-transcoder.prefs file to prefs (see below) and place it in ~/.channels-transcoder/ or  ~/Library/Application Support/channels-transcoder/ (Mac default).  Unless you use the install script, you should edit the first line of the prefs file (DEST_DIR) and make sure that Plex is set up to that it points at "$DEST_DIR/Movies" and "$DEST_DIR/TV Shows".  There are instructions in the prefs file next to each option that can be set (see section below on "Preferences file").

On this first run (or install script) it will also initialise a database which lists previously transcoded recordings, but note that it will by default not transcode any previously recorded shows unless you respond to the DAYS prompt when asked (install script) or you run from the command-line with these options:

channels-transcoder.sh CLEAR_DB=1 DAYS=N

where N is the number of days backlog you want clearing (so e.g. DAYS=7).  This will reset the database and mark all previously recorded shows (before N days ago) as having already been transcoded.  This may take a long time, depending on your system and how much stuff you have.  DO NOT run transcode-plex.sh again until this is complete.  The install script will handle most of this for you.

**Subsequent runs**

From now on, channels-transcoder.sh should work fine if run on a repeated cycle, e.g. every 24 hours.  If an existing instance of it running is found, or if Channels DVR is recording or comskipping, it will wait up to a user-defined amount of time (default=82800 seconds or 23 hours) before executing.  If more than one existing instance is found, channels-transcoder.sh will exit immediately. Note that HandBrakeCLI h.264 encoding is well optimized for multiple CPUs/threads, and so most of the time there is little benefit to running it multiple times.

Setting BUSY_WAIT=0 will prevent waiting for Channels DVR or channels-transcoder activity to cease, although it will still quit if 2+ instances of channels-transcoder are found.  

If you want something specific transcoded ASAP, and have no concerns about resources, you can override this behaviour with BUSY_WAIT=0, typically in conjunction with DAYS=0. 

**Preferences file**

The preferences file mentioned previous has a lot of settings.  This might seem intimidating, but most are not needed for regular users, and all are commented extensively within the file.  The only critical one for MOST users is the DEST_DIR one, which points at somewhere Plex can see.  The install script will prompt you for it.  It is assumed that "TV Shows" and "Movies" are subdirectories in that file.  Once set up, you should easily be able to add these folders to Plex.  If you prefer to integrate with existing Plex folders, and your "TV Shows" and "Movies" folders are named or configured in that way, you can work around it using symbolic links.

Some interesting options are CHAPTERS=1 (selected by default), which uses Channels DVR commercial markers as chapter markers in the output file, and COMTRIM=1, which actually completely removes the commercial breaks.  I do not recommend  using this latter mode unless you are very confident in the commercial detection, which in my experience produces quite a few blunders unless you have tuned your comskip.ini file extremely carefully. Note that if both are set, COMTRIM will "win".

By default, PRESET="AppleTV 3".  This is suitable for most modern devices, but if it doesn't work for you just change it, or over-ride from the command line (see below). Note that this is technically an obsolete preset, which is used for backward compatibility with versions of HandBrakeCLI that are still often distributed with Linux systems.  The same settings can also be had using the more up-to-date "Apple 1080p30 Surround" preset.  If you really care about preserving 60 Hz source data, you could try "Apple 1080p60 Surround" on recent (>=1.0) versions of HandBrakeCLI, but you might be liminting what devices can play it back (Apple TV 4, iPhone 6/7, iPad Pro will work). If your devices struggle with those suggestions, "AppleTV 2" or (for more recent versions of HandBrake) "Apple 720p30 Surround" will be better, but you will not get 1080 resolutions.  Note that file sizes might end up on average slightly larger for the same resolutions with these less capable presets, as they restrict what compression tricks can be used.

If you would like something more suitable for limited upload bandwidth, I recommend using the MAXSIZE setting (e.g. MAXSIZE=720 for 720p; 576 is the lowest I would personally go for decent quality), which reduces resolution and substantially reduces filesize.  You could also lowering the speed to e.g. veryslow, which supposedly trades speed to processing time, but in reality these slower settings do not always produce smaller files.

Note that transcoding is done in software, and so will be a CPU hog on most systems, and thus it's worth running with "nice" set (default is 10, 0 is normal priority, 19 is lowest).  It should be possible to edit the script to use hardware transcoding if desired, but I haven't tested that yet (please contact me if you're interested).  I have attempted to balance output quality, file size and processor load so that it will work well for most end-users.

Finally, some "bonus" feature described below utilize IFTTT for phone notifications, GNU parallel for offloading transcoding to other machines, and more.  Some of these are documented below.

**Command line operation**

All of the default options within the script can be substituted for on the command line. In its most basic mode, simply run:

channels-transcoder.sh

from the command line and it will scan the source directory (the "TV" folder where Channels DVR stores its recordings).

If you would like to overload any of the options above, simply add them as arguments, e.g.

channels-transcoder.sh DAYS=1 MAXSIZE=540 COMTRIM=1

will only search for files created in the last 6 hours (360 minutes) and will create smaller 540p files with commercials trimmed. Note that it will not transcode previously finished shows until you re=initate your database (CLEAR_DB=1). Also, it should be noted that the arguments are case-sensitive.

An additional option for command line execution only is to specify specific recordings on the command line.  This is done without the VAR="parameter" format, and can use either a part of the filename, or the specific Channels DVR recording ID, e.g.:

channels-transcoder.sh 11 12 14
channels-transcoder.sh "Sherlock"

The OVERWRITE=1 option forces it to copy over previous versions, which by default it will not do.  The numerical version is explicit, whereas the latter is a search expression on the filename, and so in this instance it would find ALL shows containing the search term Sherlock.  You can be as specific as you like with the search expression, and so the entire filename can be given to avoid ambiguity.  However, if you choose to search by directories, note that these are relative to the root of the Channels DVR database, so "Sherlock/Season 1" would work, but "DVR/TV Shows/Sherlock/Season 1" would not.  Finally, it should be noted that in both of these instances it will not check if previously transcoded in the database.

Given that there are limits to the number of instances (2) of channels-transcoder.sh that will run, there are circumstances under which running this manually may prevent automatic execution.

If, for whatever reason, you wish to reset your transcode database, you can always do so as per initial setup instructions, e.g. channels-transcoder.pl CLEAR_DB=1 DAYS=N.


**Daemon/cron management**

This should be set up for users that run the install script.

For most Linux users it's probably easiest to run this as a cron job, e.g. the default:

1 0 * * * /usr/local/bin/channels-transcoder.sh > ~/.channels-transcoder/log

This would running at 12:01am every night (by default the script will also wait internally for up to 4 hours for Channels DVR to stop recording/comskipping before starting).

For Mac users, I've included a LaunchAgent file in this archive (com.getchannels.channels-transcoder.plist), typically placed into the ~/Library/LaunchAgents directory. Once it's there, run the following:

sudo launchctl load /Library/LaunchAgents/com.getchannels.channels-transcoder.plist
sudo launchctl start com.getchannels.channels-transcoder

The log file is also in the prefs director, and so can be monitored easily (e.g. tail -f ~/.channels-transcoder/log).

I'm working on more daemon and file monitoring approaches, and would appreciate additions from others.

**Phone notifications**

This is a convenient way to monitor your jobs. Unfortunately IFTTT are restricting the ability to share applets at the moment, for anyone other than developers, but it's fairly easy to roll your own.

You will need to set up an IFTTT account and the app installed on your phone. Then you should add the Notifications (https://ifttt.com/if_notifications) and Maker (https://ifttt.com/maker) services, before going to Maker settings (linked from https://ifttt.com/maker), copying the 22-digit code at the end of the URL under Account Settings and adding it to the IFTTT_MAKER_KEY variable in the script. Finally, you'll set up an IFTTT applet thus as per the graphic (ifttt-maker-transcode-plex.png).

**Parallelization**

I have been experimenting with GNU parallel, giving the ability to (i) farm your processes out to other computers, and (ii) wait for available resources before encoding. It looks to be working fairly well, and so I added it as a feature, which is by default turned off. However, IF you want to give it a go, be my guest.  Simply add explicit reference to the PARALLEL_CLI entry under prefs, and set your -S (server) options within the PARALLEL_OPTS setting.  You might want to try running with BUSY_WAIT=0, but there are some (admittedly unlikely) circumstances under which multiple instances might end up transcoding the same file multiple times.

I do not recommend parallelizing between cores on a single machine (i.e. setting -j to more than 1), because (i) Handbrake with x264 is very scalable between cores already, and so even though you might see a marginal potential gain, there are plenty of other bottlenecks that could reverse that gain, and (ii) You'll actually see your files later on average, because of non-sequential delivery.

*Parallelisation Requirements:*

i) A RECENT version of GNU parallel installed both on this machine and any others you wish to send the commend to.
ii) The DEST_DIR must be visible in the same location on your drive on your remote system. This will involve drive mounting using NFS, AFP or SMB. I do not recommend SMB due to erratic file access. I also do not recommend trying this unless you have very smooth Gigabit networking or better. I'll work on different ways to implement this in the future that might be more efficient.
iii) Passwordless logins set up with ssh-keygen for remote ssh sessions on target machines.
iv) To read documentation on GNU parallel. This is not for beginners, and you will need to set up your system correctly to be able to use it.

Note that additional options (which can be edited in PARALLEL_OPTS) have been added to GNU parallel over the years, and at least one of those, specifically --memfree, I use. Please either update to a 2016+ version or, if you have problems, delete the "--memfree 700 M". (Note that my own tests have shown that the default settings {PRESET="AppleTV 3", SPEED="veryfast", MAXSIZE=1080} have shown that 700 MBytes is about right if the source is 1080p, and 425 Mbytes if the source is 720p. These things are best to tune for yourself.)

**Other scripts**

Over time I will be adding other scripts to this archive.
