---
layout: post
title: Installing Datomic transactor with launchd
---

# {{page.title}}

In a recent project I had the need for automatically starting a Datomic transactor when booting a Mac server. On Mac OS systems and some other Linux/BSD systems, starting and stopping programs is handled by the `launchd` program.

In this post I will briefly describe the steps needed for setting up a Datomic transactor with `launchd` to ensure that it is automatically started when booting the system.


## Downloading Datomic

Datomic comes in a free and a commercial version. Here I will assume you have downloaded the free version from [http://www.datomic.com](http://www.datomic.com) into the `~/Downloads` folder on your system.

In a terminal, create an appropiate place for your Datomic installation.

{% highlight bash %}

sudo mkdir -p /usr/local
cd /usr/local
sudo unzip ~/Downloads/datomic-free-0.8.3591.zip
sudo ln -s datomic-free-0.8.3591 datomic

{% endhighlight %}


## Setting up a user

The Datomic process should run under a restricted privilege user. I recommend creating a dedicated user and group for this purpose. First, check that the group id and user id 109 is free on your system. If it is not, use another free id. You can check the numbers using `dscl`

{% highlight bash %}

dscl . -list /Groups PrimaryGroupID | grep 109
dscl . -list /Users  UniqueID | grep 109

{% endhighlight %}

The following commands will create the group `_datomic` and the user `_datomic`

{% highlight bash %}

sudo dscl . -create /Groups/_datomic PrimaryGroupID 109
sudo dscl . -create /Groups/_datomic RealName       "Datomic Users"
sudo dscl . -create /Groups/_datomic Password       \*
sudo dscl . -create /Users/_datomic  UniqueID       109
sudo dscl . -create /Users/_datomic  PrimaryGroupID 109
sudo dscl . -create /Users/_datomic  HomeDirectory  /usr/local/datomic
sudo dscl . -create /Users/_datomic  UserShell      /usr/bin/false
sudo dscl . -create /Users/_datomic  RealName       "Datomic Administrator"
sudo dscl . -create /Users/_datomic  Password       \*

{% endhighlight %}


## Setting file permissions

We need to make sure the user and group of the Datomic installation itself is `root` and `wheel`. We also need to make sure the `_datomic` user which is going to run the process will have a place to write the logs and data from the database. In this tutorial, I have chosen to place the data and logs under `/usr/local/`. If you have a dedicated place for data on your system, such as `/opt`, you should put it there instead

{% highlight bash %}

cd /usr/local/datomic
sudo chown -R root:wheel .
sudo mkdir data
sudo mkdir log
sudo chown _datomic:admin data log
sudo chmod 2770 data log

{% endhighlight %}


## Configuring the transactor

You need to set the log and data directory in the transactor properties file. In this tutorial, I've used the template file from `config/samples/free-transactor-template.properties` in the installed Datomic system. The edited file is placed in `/usr/local/datomic/config/free-transactor.properties`.

{% highlight properties %}

protocol=free
host=localhost
#free mode will use 3 ports starting with this one:
port=4334

## optional overrides if you don't want ./data and ./log
data-dir=/usr/local/datomic/data
log-dir=/usr/local/datomic/log

#pid-file=<write process pid here on startup>

{% endhighlight %}


## Creating a plist file

`launchd` reads plist files to determine which programs to start when booting the system. Save the following pice of code in the file `/Library/LaunchDaemons/datomic.transactor.plist`.

{% highlight xml %}

<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
	<key>EnvironmentVariables</key>
	<dict>
		<key>JAVA_HOME</key>
		<string>/System/Library/Frameworks/JavaVM.framework/Home</string>
	</dict>
	<key>Label</key>
	<string>datomic.transactor</string>
	<key>ServiceDescription</key>
	<string>Datomic Transactor</string>
	<key>UserName</key>
	<string>_datomic</string>
	<key>GroupName</key>
	<string>_datomic</string>
	<key>WorkingDirectory</key>
	<string>/usr/local/datomic</string>
	<key>ProgramArguments</key>
	<array>
		<string>/usr/local/datomic/bin/transactor</string>
		<string>/usr/local/datomic/config/free-transactor.properties</string>
	</array>
	<key>RunAtLoad</key>
	<true/>
	<key>KeepAlive</key>
	<true/>
	<key>StandardOutPath</key>
	<string>/usr/local/datomic/log/launchd-stdout.log</string>
	<key>StandardErrorPath</key>
	<string>/usr/local/datomic/log/launchd-stderr.log</string>
</dict>
</plist>

{% endhighlight %}

The configuration will ensure that the transactor is started upon boot, that it is executed as the correct user, and that output will be written to the Datomic log directory. Please make sure that the locations of the Java framework etc. matches your system.

Before the configuration is used by the system, it must be loaded using `launchctl`

{% highlight bash %}

sudo launchctl load /Library/LaunchDaemons/datomic.transactor.plist

{% endhighlight %}

Use the `unload` option if you need to remove the datomic transactor.

{% highlight bash %}

sudo launchctl unload /Library/LaunchDaemons/datomic.transactor.plist

{% endhighlight %}

Since the transactor is configured to be up at all times, you can restart if by asking `launchctl` to stop it

{% highlight bash %}

sudo launchctl stop datomic.transactor

{% endhighlight %}


## Final remarks

That should be all that is needed for setting up Datomic on a Mac OS system. Please check the logs to make sure your system is booted. Apart from the log files for standard out and error defined in the plist file, `launchd` logs errors to `/var/log/system.log`.
