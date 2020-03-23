# OS Services

## Let's get to business

### Goal

In this excersize we'll create a simple service and make sure it's always up. Our service will always serve a random number, instead of a dice, which is sooo stone-age.

### Steps

1. Let's create a simple one-liner to generate a random number from 1 to 6.

   ```bash
   #! /bin/bash
   echo $(( ( RANDOM % 6 )  + 1 ))
   ```

   Write these two line into `/opt/dice/dice.sh` using your favorite tool. You may just `cat` them, use `vi` or whatever. You probably don't have this directory already existing, so create it with `mkdir -p /opt/dice`. Then make this script runable by running `chmod 755 /opt/dice/dice.sh`.

   You can test our complex program by running `/opt/dice/dice.sh` or by adding dice to your PATH, using `export PATH=$PATH:/opt/dice` and then just `dice.sh`.

1. Unfortunately, this program is only accesible to us, users of this one machine. Because this program is so useful on the one hand, but so complex that we don't want others to suffer the agonies of writing, testing and debugging it on their own machines, let's embrace the M/SOA way and make it accessible as a service, consumable through HTTP.

   To achieve this intetion we'll change thigs a bit. Our script will always run in the background and write a random number into a file. We'll run a web server that will serve this file. Than, anyone will be able to throw away his/hers dices (pun inteanded) and use a modern micro-service instead. Let's edit `/opt/dice/dice.sh`

   ```bash
   #! /bin/bash
   while true
   do
       echo $(( ( RANDOM % 6 )  + 1 )) > /var/www/html/index.html
   done
   ```

1. Before running this script again, use whichever package manager relevant for your Linux distribution to install [Apache HTTPD](https://apache.org/httpd). If you're using Fedora, like I do, run `sudo dnf install -y httpd`. After you have HTTPD installed, run `systemctl enable httpd ; systemctl start httpd`. Don't warry about it too much for now, we'll get back to that shortly.

   Now we can run our script `dice.sh` again. Notice that you don't get your terminal back. That's only to be expected and we'll get back to that. From another termial run `curl http://127.0.0.1/index.html`. Try this several times to verify your dice really is random. See? a real dice is much slower.

1. It's no fun at all having to use another terminal. Let's run `dice.sh` in the background. `dice.sh & ; curl http://localhost/index.html`. Much better. Stop dice by running `pkill dice.sh`

1. Were still quite far from perfect. Open two terminals. In the first terminal, run `dice.sh &`. In the second, run `curl http://localhost/index.html` a couple of times. Now close the first terminal and run `curl` again in the second terminal a couple of times. Do you still get random numbers?

   We don't want our micro-service to stop working whenever we excidentally close the wrong terminal. The trick to do so in Linux is by doing [double fork](https://thinkiii.blogspot.com/2009/12/double-fork-to-avoid-zombie-process.html). There are many examples of C code for that, but fortunatly, we can have this simple BASH one-liner and avoid the fuss `(dice.sh &) &`.

   Does it really solve our problem? Prove it to yourself.

1. Almost without realizing we've gone the first step into [daemonizing](https://www.commandlinux.com/man-page/man7/daemon.7.html) our process. In order for it to become a true daemon, we need to detach ourselves completely from the terminal.

   Now you probably ask yourself; but we've already proved that we were detached from the terminal. We closed the terminal and the process kept running.

   To understand what we're trying to solve, change `dice.sh` to be

   ```bash
   while true
   do
       echo $(( ( RANDOM % 6 )  + 1 )) > /var/www/html/index.html
       echo "the dice has spoken!" ; sleep 10
   done
   ```

   Run `(dice.sh &) &` again. Now it's true that if you'll close the terminal the dice will keep on rolling, but as long as the terminal is open those messages will keep on bugging us. To really detach from the terminal, and also to run `dice.sh` without worrying about sending it to the background ourselves, let's make it as follows

   ```bash
   #! /bin/bash

   # A deamon should run from the root directory.
   cd /

   # As we've said before, we're going to do a double-fork. So we'll have a child and a grand child.
   if [ "$1" != "child" ] && [ "$1" != "grandchild" ]
   then
       setsid --fork /opt/dice/dice.sh # We could have just ran `/opt/dice/dice.sh &` , but now you'll read man setsid :)
       exit 0
   fi

   if [ "$1" == "child" ]
   then
       # A daemon should reset its umask
       umask 0
       # A daemon should detach from input/output
       exec 0<&-
       exec 1>/dev/null
       exec 2>/dev/null
       # Let's run a grandchild
       /opt/dice/dice.sh grandchild &
       exit 0
   fi

   while true
   do
       echo $(( ( RANDOM % 6 )  + 1 )) > /var/www/html/index.html
       echo "the dice has spoken!"
   done
   ```

   Now try running `dice.sh` and see if we're detached enough for your liking.

1. Once we'll run `dice.sh` it will stay in the background forever and ever. Or will it? Waht if the service itself dies?

   To manage our daemons, Linux systems usually use service managers. If you're using Fedora/CentOS/RHEL, your service manager is [SystemD](https://systemd.io). To tell it to manage `dice.sh` as a service (which is quite different from aaS), we need to write a thing called [unit file](https://www.freedesktop.org/software/systemd/man/systemd.unit.html). Let's create a file `/usr/lib/systemd/system/dice.service`. Note that the path is imprtant, and so is the file suffix.

   ```ini
   [Unit]
   Description=Keep them dices rolling, man!
   After=httpd.service

   [Service]
   Type=forking
   Restart=always
   ExecStart=/opt/dice/dice.sh

   [Install]
   WantedBy=multi-user.target
   ```

   To get `SystemD` to know our new service, run `systemctl daemon-reload`. Now run `systemctl start dice`. Verify it's up.

1. To be sure the service will start at boot time, `SystemD` needs to know in which step of the boot sequence it should run. To tell `SystemD` to run it as part of the "mutli-user" step, run `ln -s /usr/lib/systemd/system/dice.service /etc/systemd/system/multi-user.target.wants/dice.service`.

   Alternatively, you can run `systemctl enable dice.service`, and it'll know in which step to start the service because of the `[install]` section of the unit file. But now you know what it really does.

1. Remember this line, `Restart=always`? Try starting the service and then killing `dice.sh`. Did `SystemD` succeeded in recognizing the process was killed?

   Because `dice.sh` is forking, it's hard for `SystemD` to follow. So let's remove all this advanced stuff we did to daemonize our script and change, in the unit file, from `Type=forking` to `Type=simple`. `SystemD` is daemonizing the process for us, but now we know what it's doing under the hood.

### Advanced

1. This guide assumes we're using `SystemD`. However, many systems use `SysV`. Read about the differences between them.
