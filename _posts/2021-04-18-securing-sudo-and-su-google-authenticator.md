---
layout: post
title:  "Securing sudo and su with Google authenticator"
date:   2021-04-18 
categories: "tech"
author: Alex
excerpt_separator: <!--more-->
asset_path: /assets/images/blog/2021-04-18/
image: 
---
<!-- tl;dr -->
Passwords suck. 
That’s why it’s usually preferred to access systems with private / public key authentication. Especially popular with ssh connections.

Nonetheless — even a private key can get compromised. It lives — most of the times — on your local machine and therefore can be exposed to other security threats like malware etc.
So let’s assume at some point our keys get compromised and an attacker is able to log in with the key to one of our servers.

Most of the times this would go unnoticed (adding a simple, yet effective monitoring solution is another article ;) ) so we have to make sure that if that happens an attacker can only do so much.

Having a good security concept in regards of user and group management is critical. On my systems, root login is prohibited and I solely work with my personal user and sudo commands.
That being the main gateway into my systems I want to lock down sudo authentication and the ability to use the su command even more.
<!--more-->

One of the easiest ways to implement was the google authenticator.
This little guide was done with the latest RHEL 8 release. 
Some of the commands may be adjusted accordingly if you use another distribution. Let’s get started.

## What you need

- The Google Authenticator app installed on your phone.
- A user with sudo privilege
- The following dependencies: `git autoconf automake make libtool pam-devel`

**Attention**: In this guide we alter authentication rules. Unless you’re confident in what you are doing, please stop.
You can potentially lock yourself out of your own system!

**During the whole installation I suggest to keep a seperate root (ssh) session open for backup purposes and to roll back any changes if it does not work as intended.**

Unless explicitly stated you need to run the command as the user with sudo privileges. The code boxes are without any prompt for easier copy & pasting ;)


## Installing the authenticator

Let‘s get the current repository from GitHub with git clone.
```
git clone http://github.com/google/google-authenticator-libpam.git
```

Change into the newly created directory and set up everything through the two scripts `bootstrap.sh` and `configure` to run the install with make and make install afterwards.

```
cd google-authenticator-libpam/
# In the directory run
./bootstrap.sh
./configure
make
sudo make install
```

This should install the google authenticator library without any input needed.

Next, let‘s configure PAM to switch over to the google authenticator for running sudo or su.

PAM is — in short — an application that manages user authentication.
<a href="https://www.redhat.com/sysadmin/pam-configuration-file">Here‘s a pretty good blog post by RedHat if you want to understand the basic structure</a>.


For this guide we will change explicitly only the files for the commands we want to add the authenticator to. 
You can also set the authenticator to be used on a higher level. 
Please check the official google documentation in the GitHub Repository on how to do that.

The two files we are going to change are found at `/etc/pam.d/sudo` and `/etc/pam.d/su/`.

Backing up the original configuration is always recommended before we start.

```
# Copy the files and change suffix to .bak (for backup)
cp /etc/pam.d/su ~/su.bak
cp /etc/pam.d/sudo ~/sudo.bak
```

Now that we have a backup of a working config in our home directory, we can safely adjust the files to our liking. If something goes wrong the backup can always be copied back and overwrite our altered configuration.

Open the file with the editor of your choice (I use vi) with `vi /etc/pam.d/sudo`
and adjust the input to look like this:

```
#%PAM-1.0
auth       sufficient   /usr/local/lib/security/pam_google_authenticator.so
auth       include      system-auth
account    include      system-auth
password   include      system-auth
session    optional     pam_keyinit.so revoke
session    required     pam_limits.so
```

As you can see we included the google authenticator pam module for the first entry with the type of „auth“.
It‘s set to sufficient because we want to return to the application if the google authentication is successful.

Next up, the su file: `vi /etc/pam.d/su`

Content of /etc/pam.d/su:
```
#%PAM-1.0
auth            sufficient      pam_rootok.so
auth            sufficient      /usr/local/lib/security/pam_google_authenticator.so
# Uncomment the following line to implicitly trust users in the "wheel" group.
#auth           sufficient      pam_wheel.so trust use_uid
# Uncomment the following line to require a user to be in the "wheel" group.
#auth           required        pam_wheel.so use_uid
auth            substack        system-auth
auth            include         postlogin
account         sufficient      pam_succeed_if.so uid = 0 use_uid quiet
account         include         system-auth
password        include         system-auth
session         include         system-auth
session         include         postlogin
session         optional        pam_xauth.so
```

Another thing to consider is to set up a timeout to sudo privileges to control after which time the authenticator code needs to be validated again.

If you want to do so, let‘s use the sudoers.d/local file as this will survive upgrades. Remember, manually added specific settings should always be done in the „.d“ folders or files.
```bash
vi /etc/sudoers.d/local
```

Add the line
```
Defaults timestamp_timeout=30
```
to set the timeout to 30min. If you want to re-authorize every sudo command with the token, set the timeout to 0.

Now run the google authenticator as your user you want to use the authenticator with.
```bash
$ google-authenticator
Do you want authentication tokens to be time-based (y/n) y
Do you want me to update your "/home/<USER>/.google_authenticator" file? (y/n) y
Do you want to disallow multiple uses of the same authentication
token? This restricts you to one login about every 30s, but it increases
your chances to notice or even prevent man-in-the-middle attacks (y/n) y
By default, a new token is generated every 30 seconds by the mobile app.
In order to compensate for possible time-skew between the client and the server,
we allow an extra token before and after the current time. This allows for a
time skew of up to 30 seconds between authentication server and client. If you
experience problems with poor time synchronization, you can increase the window
from its default size of 3 permitted codes (one previous code, the current
code, the next code) to 17 permitted codes (the 8 previous codes, the current
code, and the 8 next codes). This will permit for a time skew of up to 4 minutes
between client and server.
Do you want to do so? (y/n) y
If the computer that you are logging into isn't hardened against brute-force
login attempts, you can enable rate-limiting for the authentication module.
By default, this limits attackers to no more than 3 login attempts every 30s.
Do you want to enable rate-limiting? (y/n) y
```

You are now prompted to connect the codes with your phone app.
The alphanumeric code probably needs to copied by hand although if you have libqrencode installed you might be lucky and able to scan a QR image.

After registering the code in your app you are good to go.
Try logging off and logging in with a fresh session and issue a sudo command. You are now getting prompted to enter the verfication code before being able to run commands as sudo.

## Conclusion
We now added an additional layer of security to our system.
There are however still some flaws.

The file which genereates the secert is stored in the corresponding /home directory of that user.
This might be an additional security flaw as the secret can easily be read and / or manipulated if one has access to the home directory.

If you want to, you can change that and store the secret in a more secure way.
A simple way to do so can be found <a href="https://blog.entek.org.uk/notes/2019/07/08/secure-sudo-with-google-authenticator.html">here</a>.