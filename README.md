# Offset
Automatically process packages and scripts at logout (for 10.12 and earlier) or login window (for 10.13 and 10.14).

## Update 2023
I no longer actively maintain, use, or support this project, but I've just updated it for Python 3 in case that might be helpful to someone. I was hoping to use [the MacAdmins Python for this](https://github.com/macadmins/python/releases), but I found it was giving errors when importing the plistlib module. So I've just put in the [Munki](https://github.com/munki/munki) Python instead, which seems to work better. Feel free to use whatever version of Python 3 you want that actually works.

I've done very minimal testing of this on Ventura (macOS 13). Your mileage may vary in Ventura itself or in Sonoma and beyond.

## Backstory
Heavily based on Joseph Chilcote's [Outset](https://github.com/chilcote/outset), which processes packages and scripts at boot and login. He believed the running of logout scripts was outside the scope of his project and also had reservations about the implementation, so he suggested I could make it and call it _offset_, which is actually a great name, when you start looking at all the dictionary definitions of the word (a counterbalance, an offshoot, an actual outset still).

There was a bit of debate about what method to use for this, since there is [a workaround](http://apple.stackexchange.com/a/151492) that uses a Launch Agent that then runs trapped code when the script gets SIGINT, SIGUP, or SIGTERM. Chilcote had some concerns about that approach: that it does not respawn if killed prematurely, and that it will run the script's commands without warning the user if the Launch Agent is killed prematurely. Anecdotally, I haven't noticed any of these issues adversely affecting day-to-day functionality in my workplace, but if there's a tool that can be used by many people in various contexts, it's good to use a method that's a bit more thoroughly tested.

Apple (since 10.4, I believe) has officially deprecated, in [Login and Logout Hooks](https://developer.apple.com/library/mac/documentation/MacOSX/Conceptual/BPSystemStartup/Chapters/CustomLogin.html), the use of login and logout hooks in favor of Launch Agents. The only problem is that they don't present an actual alternative to the logout hook, and the logout hook appears to still work as of 10.12 (Sierra).

Greg Neagle suggested using a launch agent that runs at the login screen (meaning the user is logged out), and that appears to be the best (safest) way to go, so that's what this project is using.

So Offset, which can be used in conjunction with Outset if you want both scripts for login/boot/on-demand _and_ logout scripts, you can install both Outset _and_ Offset. Of course, you can also write custom Launch Agents and Launch Daemons for every single script, but the point here is convenience--placing scripts in a few folders to run instead of creating separate launchd's for each script.

## Technical Notes on Implementation
* In addition to slightly modifying a few variable names, I significantly pared down Joseph Chilcote's original script, because I couldn't come up with any use cases for a logout-once scenario. I figured anything you wanted to do once you could do with a login-once using Outset instead of a logout-once in Offset. Logout scripts typically clean something up every time a user logs in. A login script that runs once per user usually sets up something as a default, which the user can later change.
* If you have only one script to run at logout, you may want to consider using Apple's deprecated LogoutHook instead of using Offset, because the direct linking will give you access to the $1 variable (currently-logged-in user). With Offset, your scripts will have to figure out the last-logged-in user from the output of ```defaults read /Library/Preferences/com.apple.loginwindow lastUserName```

### Notes about macOS 10.13 (High Sierra) and 10.14 (Mojave)
In High Sierra, the change from loggedIn to loggedOut in the com.apple.loginwindow.plist doesn't happen immediately. The time it takes to switch seems to vary. It can be a little more than a second or even slightly more than three seconds. I did some trials of having a time delay before checking for the loggedIn/loggedOut status, and there didn't seem to be a consistent time delay that would reliably work. The same appeared to be the case for 10.14. So if Offset sees the OS is 10.13 or greater, it will pretty much just run at the login window. I haven't done extensive testing on this, but I don't believe it will run twice on reboot.

### Caveats for macOS 10.12 (Sierra) and earlier
* Even though in practice, Offset will essentially run logout scripts, technically it's actually running login window scripts. There are four scenarios in which this is a bit messy, but for all practical purposes will likely not cause any harm:
 * If you reboot instead of log out, your "logout" scripts won't run until after the reboot (not technically right when the user has logged out).
 * If you shut down and then start up again, similarly, your "logout" scripts won't run until you start up and get to the login screen.
 * If you log out (e.g., 10:58:20) and _then_ reboot within the minute (e.g., 10:58:45--not sure why you would ever do this), it's possible your "logout" scripts will run twice for the same user--once when you've logged out, and then once when you've reboot. Offset does prevent the re-running of the scripts if the reboot is not in the same minute as the logout. Unfortunately, the way Mac OS X keeps track of login sessions, the timestamp is based only on the minute and not the seconds!
* Keep in mind Offset runs any scripts as root. Be careful when you're writing those scripts!

#### Unexpected Bonuses for macOS 10.12 (Sierra) and earlier
* Oddly enough, even if you have a user set to automatically log in, Offset will still work when you reboot your Mac--I've tested this!
* And if you have a user logged in locally on a machine and another user logged in simultaneously on the same machine remotely via Apple Remote Desktop, and both users log out around the same time, scripts will run both times (and the lastUserName will work in scripts for both users). I have tested this.

## How to Install Offset
### .pkg Method
Go to the [the releases page](https://github.com/aysiu/offset/releases), where you'll be able to download a .pkg file and just run it. 

### Create package yourself
Use [munkipkg](https://github.com/munki/munki-pkg) to build (and maybe sign?) the package yourself.

## How to use Offset
Put any scripts or packages in the **/usr/local/offset/logout-every** folder and make sure they have root:wheel ownership. The scripts should have 755 permissions. Packages should have 644 permissions.

## Acknowledgements
Huge shoutout to [@chilcote](https://github.com/chilcote), whose code this project (in a Frankenstein's monster way) is based on a rearrangement of. Also kudos to [@gregneagle](https://github.com/gregneagle) for suggesting the login window Launch Agent as an alternative to the logout hook or the interrupted user-context Launch Agent.
