# freebsd-sidekiq

An init script for running sidekiq on FreeBSD. Based entirely on the excellent [freebsd-puma] script by [snake66]. 
All I have done is to replace puma specifics with sidekiq specifics, and update this readme.

This rc script works with either RVM and RBENV installed in your deploy user's directory, or with a globally installed ruby.

This script runs sidekiq via the Freebsd daemon tool.
So it should be compatible with the latest sidekiq version, which dropped internal daemonizing support (https://www.mikeperham.com/2019/09/03/welcome-to-sidekiq-6.0/)


Simply place the `sidekiq` script in your `/usr/local/etc/rc.d` directory, modify it if necessary, and configure your application via variables in `/etc/rc.conf`
This has been tested on **FreeBSD 12.1**

## Make sure sidekiq starts after your database launches!

The only thing you might need to configure in the rc script is to change the `REQUIRE` line to specify your database (I use PostreSQL so that's what's in the repo)

For example, if you were using MySQL, you would change
    # REQUIRE: LOGIN postgresql 
to
    # REQUIRE: LOGIN mysql-server

You might need to add other services to this list if your Rails application requires them.


## Quick Setup

To get up and running quickly, adjust the `REQUIRE` line like above, and add edit your `/etc/rc.conf`:

For Capistrano or Capistrano-like directory layouts:

    sidekiq_enable="YES"

    # this is the path to where your application is deployed via Capistrano
    # (the parent directory of the `current` directory)
    sidekiq_directory="/u/application"


For Non-Capistrano-like layouts:

    sidekiq_enable="YES"
    sidekiq_command="/u/application/bin/sidekiq"
    sidekiq_pidfile="/u/application/tmp/pids/sidekiq.pid"
    sidekiq_config="/u/application/config/sidekiq.yml"
    sidekiq_chdir="/u/application"
    sidekiq_user="deploy"

    # Uncomment this if using a different RAILS_ENV/RACK_ENV than production
    #sidekiq_env="staging"


## Starting/Stopping/Restarting and Upgrading sidekiq

You can now start sidekiq like any other FreeBSD service:

    /usr/local/etc/rc.d/sidekiq start

There's also a handy `show` command to look at your final sidekiq configuration:

    /usr/local/etc/rc.d/sidekiq show

To prepare shutdown, you can send a quiet (prestop) command. 
(kill TSTP) https://github.com/mperham/sidekiq/wiki/Deployment
Note: I tried to name it quiet, but that is used by FreeBSD internally?

    /usr/local/etc/rc.d/sidekiq prestop

And when you're done riding sidekiq, you can shut it down

    /usr/local/etc/rc.d/sidekiq stop


## `/etc/rc.conf` Details


### Using a Capistrano directory layout

The rc script does as much as possible to help you out. If you are using Capistrano, or a Capistrano-like directory structure, then you can just specify the directory of your application (the parent directory of `current`):

    sidekiq_enable="YES"
    sidekiq_directory="/u/application"

This infers all sorts of information about your app (you can always run `/usr/local/etc/rc.d/sidekiq show` to see what your configuration is. **Note** the variable names listed here are without the leading `sidekiq_` prefix that you would need to specify in `/etc/rc.conf`):

    #
    # sidekiq Configuration for application
    #

    command:        /u/application/current/bin/sidekiq
    command_args:   
    pidfile:        /u/application/shared/tmp/pids/sidekiq.pid
    config:         /u/application/current/config/sidekiq.yml
    log:            /u/application/current/log/sidekiq.log
    init_config:    /u/application/current/.env
    bundle_gemfile: /u/application/current/Gemfile
    chdir:          /u/application/current
    user:           deploy
    nice:           
    env:            production
    flags:           -e production -C /u/application/current/config/sidekiq.yml

    start_command:

    su -l deploy -c "export BUNDLE_GEMFILE=/u/application/current/Gemfile && . /u/application/.env && cd /u/application/current && /usr/sbin/daemon -p /u/application/tmp/pids/sidekiq.pid -o /u/application/current/log/sidekiq.log /u/application/current/bin/sidekiq -e production -C /u/application/current/config/sidekiq.yml"

Let's look at these settings one by one:

`command`: By default, it uses the `current/bin/sidekiq` [bundler binstub][binstub] located in your project to ensure your gems are loaded. `command` comes from FreeBSD's `rc.subr` init system functions.

`command_args`: This is the standard FreeBSD's `rc.subr` variable that holds the arguments to the above `command`.  Typically you don't need to set this.

`pidfile`: This is also part of FreeBSD's `rc.subr` system. This is where the built in functions will look for the pid of the process. By default, this rc script looks in the `current/tmp/pids/sidekiq.pid` file.

`config`: This is the path to sidekiq's config file where sidekiq will find it's settings. By default this rc script looks for a file called `current/config/sidekiq.yml`

`init_config`: This is a shell script file that is included in the environment before sidekiq is executed. In this file you can include `export VAR=value` statements to pass environment variables into your rails app. By default, this init script looks for a file called `current/.env` and uses that. If that file doesn't exist, this rc script will skip this functionality (as seen in the above example).

This could be used in conjunction with [dotenv][dotenv] in development since dotenv accepts lines beginning with `export`


`bundle_gemfile`: This is the path to the `Gemfile` of your project. This rc script sets the `BUNDLE_GEMFILE` environment variable to this value. By default it looks to `current/Gemfile`. This is required so that sidekiq uses the most current `Gemfile` (rather than the one in the specific deployment directory) when an upgrade is performed.

`chdir`: This is the directory we `cd` into before running sidekiq. By default it's the currently deployed version of your application "`current/`"

`user`: This is the user that sidekiq will be run as. By default we do like [Passenger][passenger] and use the owner of the `sidekiq_directory`.

`nice`: The `nice` level to run sidekiq at. Usually you'll leave this alone.

`flags`: This is a variable defined by FreeBSD's `/etc/rc.subr` init system, and contains the flags passed to the command (sidekiq in this case) when run. This variable is built up from the variables above, but you could manually specify `sidekiq_flags` in your `/etc/rc.conf` to override them.

`start_command`: Here you can see the full command that will be run when you start this service. It's a beaut' isn't it?

You can override any of these parameter in your `/etc/rc.conf` by simply specifying the variables like you see below (you can pick and choose which to override).


### Using a custom directory layout

Using your own layout is easy, you can just leave the `sidekiq_directory` variable out of your `/etc/rc.conf` and specify all of the above variables manually. Here's a list of those variables for your convenience:

    sidekiq_command:      The path to the sidekiq command
    sidekiq_command_args: The non-flag arguments passed to the above command.  Typically you do not need to set this.
    sidekiq_pidfile:      The path where sidekiq will put its pid file
    sidekiq_config:       The path to the sidekiq config file
    sidekiq_log:          The path to the sidekiq log file
    sidekiq_chdir:        The path where this script will `cd` to before starting sidekiq
    sidekiq_user:         The user to run sidekiq as
    sidekiq_nice:         The `nice` level to run sidekiq as. Leave blank to run un-niced
    sidekiq_env:          The RAILS_ENV (or RACK_ENV) to run your application as. (default: production)
    sidekiq_flags:        The flags passed in to sidekiq when starting (not counting the sidekiq_command_args specified above). Override this for complete control of how to start sidekiq.


### Deploying multiple applications

This is all find and dandy, but some of you might have multiple applications running on the same server (even if it's just a staging and production version of your app).

This rc script can work with multiple applications. It works similarly to how postgresql's rc script works on FreeBSD.

You simply specify your profiles in your `/etc/rc.conf` with the `sidekiq_profiles` variable. This is a space separated list of application names.

Then you can customize each application by specifying variables in this form:

    sidekiq_<application-name>_variable=VALUE

Here's a simple example (I can leave the _env variable out of the production declaration since it's the default value)

    sidekiq_enable="YES"
    sidekiq_profiles="application_staging application_production"

    sidekiq_application_staging_enable="YES"
    sidekiq_application_staging_directory="/u/application_staging"
    sidekiq_application_staging_env="staging"

    sidekiq_application_production_enable="YES"
    sidekiq_application_production_directory="/u/application_production"

You can use the simplified Capistrano `directory`-based configuration like above, or you can specify all of the variable's separately, for a fully custom setup

### Customizing the script

If you want to customize the default, calculated values you want to look in the `_setup_directory()` function. This is what is called when you specify that you want to use a Capistrano-like directory layout by specifying `sidekiq_directory` or `puma_<application-name>_directory` in your `/etc/rc.conf`.

If you use a different deployment strategy than Capistrano, you could adjust the default values to work with your system.



[freebsd-puma]: https://github.com/snake66/freebsd-puma
[dotenv]: https://github.com/bkeepers/dotenv
[binstub]: https://github.com/sstephenson/rbenv/wiki/Understanding-binstubs
[freebsd-unicorn]: https://github.com/caleb/freebsd-unicorn
[Caleb]: https://github.com/caleb
