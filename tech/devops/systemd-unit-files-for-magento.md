# Systemd Unit Files For Magento

## Preface

Magento gives you many ways to do many things, but everyone needs two..

1. A way to run scheduled tasks
2. A way to keep those scheduled tasks under control

There are crons, there are workers, there are queues and there are indexers.

You _can_ run everything from crontab entries that run as `magento`, but at some point, you'll likely need to manage a daemon differently than Magento wants you to. Likewise, running _everything_ from cron is a quick path to zombie processes and the eventual exhaustion of system limits (numfiles / numprocs are usually the first to go).

Enter Systemd https://en.wikipedia.org/wiki/Systemd - a much maligned but extremely competent runtime management tool that is now ubiqitous, (and pretty damn good if you can get past the traditional unix position of "one tool to do one thing").

## Systemd quickstart (very quick)

There are two things that you care about with systemd, units (.service) and timers (.timer).

Skip this bit if you know sysd already...

### Units (service)

A unit is the main "thing" you'll be creating. Every unit that we'll deal with here contains at least three parts.

1. `[Unit]` - The main structure that defines what this thing should be called and what we link it to
2. `[Service]` - This is where you define how the thing runs and who it runs as
3. `[Install]` - Controls what the thing does in the context of the system

(There are many _much_ better explanations around, but this is what I'm using here)

The particular type of unit we're working with here will be `.service` files. All services are units, but not all units are services... It's not confusing, just don't think about it too much.

### Timers (schedule)

Timers are quasi-cron tasks that operate on a schedule. You specify a timer to trigger a service, so for our scenario the magento cron will have both a Unit and a Timer. (timers are also units, but bear with me)

For this scenario, we only need one set of unit files for the magento cron.

1. `magento-cron.service` which would like like the above, with the three sections we need to run it
2. `magento-cron.timer` which would have;
   1. `[Unit]` - The main structure that defines what this thing should be called and what we link it to
   2. `[Timer]` - Your repetition and run rules go here (it gets crontab'ish here)
   3. `[Install]` - Controls what the thing does in the context of the system

## The problem statement

If you're running a stock Mage2, there's like a section in your `env.php` that looks like this.

```php
    'cron_consumers_runner' => [
        'cron_run' => true,
        'max_messages' => 1000,
        'consumers' => [
            'async.operations.all',
            'codegenerationProcessor',
            'inventoryQtyCounter',
            ...
        ]
    ],
```

ref: https://devdocs.magento.com/guides/v2.3/config-guide/mq/manage-message-queues.html

 > n.b. `--consumers-wait-for-messages=<value>` did not exist when we started on our journey. This parameter (intro'd in 2.3.4 // https://github.com/magento/magento2/issues/23540) would have helped a little bit, but I like our solution better - because it supports a feature I'll mention later on..

This works fine, in some scenarios, but it falls apart really quickly once you deviate away from an M2 deployment that isn't acting as a public endpoint, doing product-related work.

Generally, the problem is sounded by the never-much-fun "high load average" alert, which upon investigation looks something like...

```bash
[user@scary-production-host ~]$ sudo -u magento ps fx | grep php | wc -l
147
```

...crap

```bash
[user@scary-production-host ~]$ sudo -u magento ps fx
   PID TTY      STAT   TIME COMMAND
130760 ?        Ss     0:00 /bin/sh -c /usr/bin/php /var/www/m2/curversion/bin/magento cron:run 2>&1 | grep -v "Ran jobs by schedule" >> /var/www/m2/curversion/var/log/cron.log
130764 ?        S      0:00  \_ grep -v Ran jobs by schedule
127522 ?        Ss     0:00 /bin/sh -c /usr/bin/php /var/www/m2/curversion/bin/magento cron:run 2>&1 | grep -v "Ran jobs by schedule" >> /var/www/m2/curversion/var/log/cron.log
127526 ?        S      0:00  \_ grep -v Ran jobs by schedule
105188 ?        Ss     0:00 /bin/sh -c /usr/bin/php /var/www/m2/curversion/bin/magento cron:run 2>&1 | grep -v "Ran jobs by schedule" >> /var/www/m2/curversion/var/log/cron.log
105192 ?        S      0:00  \_ grep -v Ran jobs by schedule
.
.
.
130797 ?        S      1:45 /usr/bin/php /var/www/m2/curversion/bin/magento queue:consumers:start async.operations.all --single-thread --max-messages=10000
118846 ?        S      1:46 /usr/bin/php /var/www/m2/curversion/bin/magento queue:consumers:start async.operations.all --single-thread --max-messages=10000
101269 ?        S      1:52 /usr/bin/php /var/www/m2/curversion/bin/magento queue:consumers:start async.operations.all --single-thread --max-messages=10000
 89893 ?        S      1:52 /usr/bin/php /var/www/m2/curversion/bin/magento queue:consumers:start async.operations.all --single-thread --max-messages=10000
```

...crappycrapcrap

As with all _really_ bad things, it escalates slowly, monotonically, like the war drum of impending doom. And this is production, so you can't get away with a solution that involves scheduling a task to kill all the dupes.

## Systemd saves the world

.. Not really, but it's pretty damn good at what it does, (which is almost everything) ..

First, we wrestle the workers away from Magento..

```php
    'cron_consumers_runner' => [
        'cron_run' => false,
        'max_messages' => 1,
        'consumers' => [
        ]
    ],
```

Then we create a template, (this makes up the majority of our systemd logic), that looks like this..

```bash
[Unit]
Description=Magento Consumer %CONSUMERLABEL%
PartOf=magento-worker.service
After=magento-worker.service

[Service]
ExecStart=/var/www/m2/bin/magento queue:consumers:start %CONSUMERLABEL% --single-thread --max-messages=10000
Restart=always
RestartSec=60
StartLimitInterval=0
User=magento

[Install]
WantedBy=magento-worker.service
```

And then, to generate the list of things we needed, we used a special script that parsed the output of `./bin/magento queue:consumers:list` and mashed it into a set of `<name>.service` files. First iteration of the "omgemergencyfixnow" script looked like this, (it's better now, but that one's still super secret).

```bash
#!/bin/sh

for SERVICE in ${./bin/magento queue:consumers:list}; do
    NICENAME=$(echo ${SERVICE} | sed -e 's,\.,-,g')
    sed -e "s,%CONSUMERLABEL%,${SERVICE},g" systemservice.template > ${NICENAME}.service
done
```

The output was a collection of unit files that we could start, stop, monitor _and_ manage indivudually. Listed out, we ended up with these;

* `async-operations-all.service`
* `codegeneratorProcessor.service`
* `exportProcessor.service`
* `inventoryQtyCounter.service`
* `product_action_attribute-update.service`
* `product_action_attribute-website-update.service`
* `quoteItemCleaner.service`

At this point, we can ship those files off to the designated processing server, run `systemctl enable async-operations-all.service` and we are covered until we get a proper solution in place.

### Breaking down the .service unit file

Systemd has so many features - it's hard to keep up, but we're using three primary patterns in these system files.

#### Heirarchy

Within the `[Unit]` section...

```bash
PartOf=magento-worker.service
After=magento-worker.service
```

and the `[Install]` section...

```bash
[Install]
WantedBy=magento-worker.service
```

We make reference to `magento-worker.service`. You may or may not know, but orchestrating tasks amongst multiple servers / instances / environments is a monumental pain in the arse. Having a "main" service to start and stop makes managing this process a bit easier.

```bash
[Unit]
Description=M2 Workers

[Service]
# The dummy program will exit
Type=oneshot
# Execute a dummy program
ExecStart=/bin/true
# This service shall be considered active after start
RemainAfterExit=yes
User=magento

[Install]
# Components of this application should be started at boot time
WantedBy=multi-user.target
```

Once you've installed (`systemctl enable magento-worker.service`) this particular unit, you'll have relatively clean and easy access to _all_ the other consumers that are installed, on this server.

If you `systemctl stop magento-worker`, systemd will chase down all the Unit files that are `PartOf=magento-worker.service` and stop them too. Ditto for `start`, `restart`, etc.

#### Actually running things

`[Service]` is your workhorse here.

```bash
[Service]
ExecStart=/var/www/m2/bin/magento queue:consumers:start async.operations.all --single-thread --max-messages=10000
Restart=always
RestartSec=60
StartLimitInterval=0
User=magento
```

Breaking this down reveals there isn't really much logic involved, (systemd is convoluted, but not really that complicated), we are just twiddling knobs and tuning runtimes to suit our desired behaviour;

* `ExecStart` - What command should be run
* `Restart` - We've chosen `always`, because we want this thing to restart regardless of whether it exited nicely, or got grumpy and threw a tantrum
* `RestartSec` - Essentially a back-off timer. We don't really want consumers cycling instantly (if there is work to do, it'll get done every _this many seconds_)
* `StartLimitInterval` - Think of this as your safety guard. We've turned it off because we do _not_ want systemd deciding the command is misbehaving. Our monitoring will detect rapid executions and we can deal with it ourselves TYVM.
* `User` - whoever this command should run as.

Do keep in mind that you should adjust these rules to suit your use case. The `RestartSec` can make things feel like you're scheduling commands, but you're not. If a process runs for 10 seconds and exits, it will start again one minute later - meaning you get a rolling window of execution, followed by a defined gap when sysd is waiting to start it again.

### Timers (bye bye crontab)

After achieving service files, we decided to address our remaining orchestration pain point. The magento cron..

Yes, there is crontab.
Yes, it's very good at executing things on a schedule
Yes, it is well known and reliable and exhibits all the good behaviour of a trusted unix utility.

But orchestrating it _sucks_! More on this particular facet later - I'll explain the solution before I detail the problem

#### What is a timer

A timer is a unit file that tells systemd to trigger a service, on a defined schedule.

First, we start with a `magento-cron.service`

```bash
[Unit]
Description=Magento Cron

[Service]
ExecStart=/usr/bin/php /var/www/m2/bin/magento cron:run 2>&1 | grep -v "Ran jobs by schedule" >> /var/www/m2/var/log/cron.log
User=magento
```

...simples...

Then we couple it with a `magento-cron.timer`

```bash
[Unit]
Description=Timer for Magento Cron

[Timer]
OnBootSec=300
OnUnitActiveSec=1m

[Install]
WantedBy=multi-user.target
```

I'll talk about `[Timer]` first.

 * `OnBootSec` Trigger the service 5 minutes after the server starts, (because it'll probably start _busy_)
 * `OnUnitActiveSec` And repeat the trigger every minute after that

This sounds like it'd be perfect, but it's not.. `OnUnitActiveSec` will from a system timer, which can slip around, meaning you won't necessarily get that perfect crontab tic-toc.

What we're after is `OnCalendar`. This uses a syntax more relatable to crontab, (not that anyone can ever remember that either...). That leaves us with a .timer that looks like this.

```bash
[Unit]
Description=Timer for Magento Cron

[Timer]
OnBootSec=300
OnCalendar=*-*-* *:*:00

[Install]
WantedBy=multi-user.target
```

This will run it every minute.

Install them, and start the timer..

```bash
[user@dev system]$ sudo systemctl enable magento-cron.*
[user@dev system]$ sudo systemctl start magento-cron.timer
[user@dev system]$ sudo systemctl list-timers --all
NEXT                          LEFT     LAST                          PASSED    UNIT                         ACTIVATES
Mon 2020-03-09 21:39:00 AEDT  10s left Mon 2020-03-09 21:38:00 AEDT  48s ago   magento-cron.timer           magento-cron.service
```

Give it a couple of minutes and check...

```bash
[user@dev system]$ sudo systemctl list-timers --all
NEXT                          LEFT     LAST                          PASSED    UNIT                         ACTIVATES
Mon 2020-03-09 21:41:00 AEDT  54s left Mon 2020-03-09 21:40:00 AEDT  5s ago    magento-cron.timer           magento-cron.service
```

Success!

## Why all this work for something so simple

A couple of reasons;

1. We can apply this approach across all environments, (local vagrant, dev, uat, preprod, prod, dr, etc, etc).
2. Systemd is easily controlled by platform management utils, (ansible, puppet, etc, etc).
3. Having systemd running services _and_ crons, ameans there's only one place to manage magento tasks.
4. Individual service files can be started and stopped as required in dev. Once you know the pattern, you know the pattern!
5. It's entirely scriptable and extensible - we can stagger services, ensure only one is running at a time, monitor everything - the list keeps going.
6. We can now incorporate monitoring to keep track of resources consumed by individual processes.
7. Crontab is now dedicated to non-application tasks.
8. We can now split crons out from magento, and control their execution, without having to manage crontabs, (we're running `./bin/magento` now, but it can just as easily run n98-magerun2).

## Systemd timers vs Crontab and other advantages

Crond is awesome, it does a great job, but when you have more than one server running magento crons, things get messy.
Likewise, when you have multiple servers that get stamped out from the same template, are deployed to in the same manner and are all available for "background" jobs, having a tool like systemd managing the runtimes is just easier.

What are now able to do is manage a catalog of services, and identify when they run, who they run as and where they run, using a system that's designed to keep the system alive. We also spare ourselves the fun and joy of dealing with lockfiles.

There is, of course, one area that I haven't covered specifically...

> What do you do during deployments, or when you need to enable maintenance mode?

This is a surprisingly easy process.

If you go back to the main service file, you'll see `ExecStart=`.
We can bound that process to detect maintenance mode using `ExecStartPre=`

```bash
[Unit]
Description=Magento Cron

[Service]
ExecStartPre=test ! -f /var/www/m2/var/.maintenance.flag
ExecStart=/usr/bin/php /var/www/m2/bin/magento cron:run 2>&1 | grep -v "Ran jobs by schedule" >> /var/www/m2/var/log/cron.log
User=magento
```

This will work for a single server under maintenance, but it won't work for multiple servers, when only one is under maintenance.

Likewise, the reverse is true, you can set up a service to _only_ run when a specific file does not exist;

```bash
[Unit]
Description=Magento Maintenance Alert
PartOf=magento-worker.service
After=magento-worker.service

[Service]
ExecStartPre=test -f /var/www/m2/var/.maintenance.flag
ExecStart=/usr/local/bin/maintenance_duration_monitor
Restart=always
RestartSec=10
StartLimitInterval=0
User=magento

[Install]
WantedBy=magento-worker.service
```

Lots of options, lots of capabilities

## References and links



In no particular order....

1. https://wiki.archlinux.org/index.php/systemd
2. https://www.digitalocean.com/community/tutorials/understanding-systemd-units-and-unit-files
3. https://stevenwestmoreland.com/2017/11/renewing-certbot-certificates-using-a-systemd-timer.html
4. http://alesnosek.com/blog/2016/12/04/controlling-a-multi-service-application-with-systemd/
5. https://wiki.archlinux.org/index.php/Systemd/Timers
6. https://fedoramagazine.org/systemd-unit-dependencies-and-order/


