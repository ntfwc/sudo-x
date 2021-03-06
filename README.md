#Description
sudo-x is a script for easily opening a graphical application as a different (usually non-root) user. It comes in a version that uses xhost and one that uses xauth. You will have to check to see which works with your system (e.g. xhost on Ubuntu 12.04, xauth on Debian 7).

sudo-x-a adds in audio support. Currently this is PulseAudio only, accomplished by simply pointing PulseAudio towards localhost as the server.

##Usage

>sudo-x user command [command parameters]

>sudo-x-a user command [command parameters]

##Rational
Linux user accounts are a powerful method for restricting software to minimum priviledge. It is fairly easy to utilize other user accounts for non-graphical programs such as servers, but when it comes to graphical applications it isn't obvious, except if the target user is root. And using a separate user account incurs minimal performance impact, which is important for software such as web browsers and video games.

##Security Considerations
X11 access still grants a fair bit of priviledges to a program. For example, it can hear your keypresses, read your clipboard, and watch your screen. And, of course, user account protections will fail in the event of a priviledge escalation.

-----------------

#Setup
##Basic
First you need a non-root user. You probably don't want this to be a login user, instead you should use a system user that will be treated as a sort of sub-account. And it will probably need a home directory, especially if you are using xauth.

>useradd -r -m [sub-user]

Now that we have a sub-user, we should give ourselves access to it. Usually this means removing password requirements in sudo for that user and adding ourselves to its group. 
Note: Being added to a new group won't take effect until your next log-in, so you may wish to log out and back in afterwards.

>adduser [me] [sub-user]

>visudo -f /etc/sudoers.d/[sub-user]-rules

			[me] ALL=([sub-user]) NOPASSWD: ALL

Now, once you have a sudo-x script, make sure the TMP_FOLDER variable in it is set to your liking. This folder will be used for locks. By default this is /dev/shm, which is usually an in-memory folder. Use chmod +x on the script and let's try opening a text editor like leafpad:

>[Path/To/Script]/sudo-x [sub-user] [editor]

Make sure to try the other version if this doesn't work. If your chosen editor pops up, you are good to go. And if you try opening a file from the editor, you should see you are restricted to what the user has access to. You can then install the script somewhere in your path, usually in /usr/local/bin/

##Audio
Currently this requires PulseAudio. You have to enable local network access on your desktop PulseAudio server. I suggest you use paprefs for this task. With paprefs open, you simply go to the Network Server tab and tick the "Enable network access to local sound devices". Now, if sudo-x is correctly installed, sudo-x-a should be able to act the much the same, except with audio.

##Graphics Acceleration
If the application you want to run requires graphics acceleration, you will need to add the user to a group that grants access to it, which is usually the video group.

>adduser [sub-user] video

##Security Improvements
###File Access
By default the umask for file access can be rather permissive. Usually this allows other users to read your files, but not write over them. This might be fine, but you may prefer that these sub-users cannot see any of your files. You can change the umask to 0007 so by default files are created without any permissions for other users. And you can take away these permissions on existing files with chmod o-rwx [files] and chmod -R o-rwx [folders]. It is also sufficient to do chmod o-rwx [folder] (no recursion) with a top-level folder, like your home directory.

###Setuid Programs
setuid programs are run as the owner, which is usually root. This also includes programs like su and pkexec, which, like sudo, allow you to change users. This is something only root can do. These user changing tools, in particular, are something you should consider restricting access to, since priviledge escalation could be just a password away, and, as mentioned above, open X11 applications can hear your key presses. You can accomplish this by changing the groups of these program files to one like sudo or admin and then making the permissions more restrictive. Use something like dpkg-statoverride if you can, so that the change will stick around if the file is updated.

		>ls -l /bin | grep su
		-rwsr-xr-x 1 root root su
		>sudo dpkg-statoverride --update --add root sudo 4750 /usr/bin/pkexec
		>ls -l /bin | grep su
		-rwsr-x--- 1 root sudo su
	

###Network
One of the wonderful things about user accounts in Linux is that they can have their own user specific firewall rules. There are several simple firewall tools, but you may need something a bit lower-level to take advantage of this. Ferm is a good choice. This allows you to set up your rules a bit like a program, and it makes user specific rules pretty intuitive. For example, here is a snippet of an OUTPUT chain rule that blocks network access, with the exception of the local PulseAudio port, for a few users:

	mod owner uid-owner (user1 user2 user3)
	{
		#Allow only pulseaudio server access
		outerface lo proto tcp dport 4713 ACCEPT;
		REJECT;
	}

-----------------

#Examples
##Web browser (Firefox)
First we set up our user:

		sudo useradd -r -m browser
		sudo adduser ntfwc browser
		sudo visudo -f /etc/sudoers.d/browser-rules

			ntfwc ALL=(browser) NOPASSWD: ALL

		sudo adduser browser video

Then try it out:

		sudo-x-a browser firefox

After logging out and back in, We should give ourselves the ability to move and delete files from the Download folder:

		cd /home/browser
		sudo -u browser chgrp ntfwc Download
		sudo -u browser chmod g+rw Download

Then we can optionally package this in its own script, where we can apply memory contraints:
		
		#!/bin/sh

		MB_LIMIT=2048
		KB_LIMIT=$(expr $MB_LIMIT \* 1024)

		#apply limits
		ulimit -v $KB_LIMIT

		sudo-x-a browsers firefox "$@"

And finally set it as the preferred application for web in the desktop environment.

##Video Games
For this use, I would suggest making user accounts for different catagories of restriction. So, for example, we can make a user for offline games called "game-jail" and one for games that need internet access called "game-jail-n". You can also have dedicated users for particular game distribution platforms.

So let's set up our two users, (which should both need video acceleration):

		sudo useradd -r -m game-jail
		sudo adduser ntfwc game-jail
		sudo adduser game-jail video
		sudo useradd -r -m game-jail-n
		sudo adduser ntfwc game-jail-n
		sudo adduser game-jail-n video
		sudo visudo -f /etc/sudoers.d/game-jail-rules

			ntfwc ALL=(game-jail) NOPASSWD: ALL
			ntfwc ALL=(game-jail-n) NOPASSWD: ALL

Then we should apply network restrictions in our firewall (in this case, using ferm):

		mod owner uid-owner game-jail
		{
			#Allow only pulseaudio server access
			outerface lo proto tcp dport 4713 ACCEPT;
			REJECT;
		}

Now we should re-login and give ourselves write access so we can create a folder in the user's home for games, which are not globally installed.

		cd /home
		sudo -u game-jail chmod g+w game-jail
		sudo -u game-jail-n chmod g+w game-jail-n
		mkdir game-jail/Games
		chgrp game-jail game-jail/Games
		mkdir game-jail-n/Games
		chgrp game-jail-n game-jail-n/Games

Make sure that, when you copy over a game, you change the group so it can be read and executed by the user. For example, if you had a game in a folder called "AwesomeGame" in your home dir:

		cd ~
		mv AwesomeGame /home/game-jail/Games
		cd /home/game-jail/Games
		chgrp -R game-jail AwesomeGame

Have fun!
		