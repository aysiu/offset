# offset
Automatically process packages and scripts at logout

## Backstory
Heavily based on (as in, reuses barely-tweaked code from) Joseph Chilcote's [Outset](https://github.com/chilcote/outset), which processes packages and scripts at boot and login. He believed the running of logout scripts was outside the scopes of his project and also had reservations about the implementation, so he suggested I could make it and call it _offset_, which is actually a great name, when you start looking at all the dictionary definitions of the word (a counterbalance, an offshoot, an actual outset still).

There was a bit of debate about what method to use for this, since there is [a workaround](http://apple.stackexchange.com/a/151492) that uses a Launch Agent that then runs trapped code when the script gets SIGINT, SIGUP, or SIGTERM. The concerns about that approach were threefold: that it runs in user space instead of root, that it does not respawn if killed prematurely, and that it will run the script's commands without warning the user if the Launch Agent is killed prematurely. Anecdotally, I haven't noticed any of these issues adversely affecting day-to-day functionality in my workplace, but if there's a tool that can be used by many people in various contexts, it's good to use a method that's a bit more thoroughly tested.

Apple (since 10.4, I believe) has officially deprecated in [Login and Logout Hooks](https://developer.apple.com/library/mac/documentation/MacOSX/Conceptual/BPSystemStartup/Chapters/CustomLogin.html) the use of login and logout hooks in favor of Launch Agents. The only problem is that they don't present an actual alternative to the logout hook, and the logout hook appears to still work as of 10.11 (El Capitan).

So this, which can be used in conjunction with Outset if you want both scripts for login/boot/on-demand _and_ logout scripts, you can install both Outset _and_ Offset. Of course, you can also write custom Launch Agents and Launch Daemons for every single script, but the point here is convenience--placing scripts in a few folders to run instead of creating separate launchd's for each script.

## Technical Notes on Implementation
.. Since the logout hook (even though it runs as root) doesn't seem to have any issues (apart from Apple officially deprecating it), that's what Offset uses.
.. In addition to slightly modifying a few variable names, I significantly pared down Joseph Chilcote's original script, because the logout hook can take only a script path and not arguments (yes, I tested this) and because I couldn't come up with any use cases for a logout-once scenario. I figured anything you wanted to do once you could do with a login-once using Outset instead of logout-once in Offset. Logout scripts typically clean something up every time a user logs in. A login script that runs once per user usually sets up something as a default, which the user can later change.
.. If you have only one script to run at logout, you may want to consider using the LoginHook directly with the script instead of using Offset, because the direct linking will give you access to the $1 variable (currently-logged-in user).
.. Keep in mind the LoginHook (which is what Offset uses) runs any scripts as root. Be careful when you're writing those scripts!

## How to Install Offset

### Easy .pkg Method
Go to the [the releases page](https://github.com/aysiu/offset/releases), where you'll be able to download a .pkg file and just run it. 

### Manual Method
Alternatively, you can manually set things up.

Create a /usr/local/offset directory and put the **offset** file in there
> /usr/local/offset/**offset**

Create a directory for **logout-every** scripts

> /usr/local/offset/**logout-every**

Make sure they're both owned by root; set to be read/write/execute, read/execute, and read/execute; and then the offset script added as a logout hook
```
sudo chown -R root:wheel /usr/local/offset
sudo chmod -R 755 /usr/local/offset
sudo defaults write com.apple.loginwindow LogoutHook "/usr/local/offset/offset"
```

## How to use Offset
Put any scripts or packages in the **/usr/local/offset/logout-every** folder and make sure they have root:wheel ownership. The scripts should have 755 permissions. Packages should have 644 permissions.
