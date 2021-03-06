<!DOCTYPE html PUBLIC "-//W3C//DTD HTML 4.01 Transitional//EN"
   "http://www.w3.org/TR/html4/loose.dtd">

<html lang="en">
<head>
	<meta http-equiv="Content-Type" content="text/html; charset=utf-8">
	<title>Instructions for configuring and running the (alpha) Twitter Realtime Plugin</title>
	<meta name="generator" content="TextMate http://macromates.com/">
	<meta name="author" content="Amy Jo">
	<link rel="stylesheet" href="./css/bright.css" type="text/css" charset="utf-8" media="screen">
	<!-- Date: 2011-01-27 -->
</head>

<body style="margin:30px;" class="bright">
    
# Instructions for configuring and running the (alpha) Twitter Realtime Plugin #

## Introduction ##

This document describes how to set up and run the currently-in-**alpha** Twitter Realtime plugin. To set it up in its current state, you'll need to be comfortable with the command line. These instructions assume that you are already familiar with how to configure and use ThinkUp.

The Twitter Realtime plugin sets up a persistent connection to the Twitter [User Streams API](http://dev.twitter.com/pages/user_streams "User Streams | dev.twitter.com"), for each active Twitter account (instance) in your ThinkUp installation, using a modified version of the [Phirehose](http://code.google.com/p/phirehose/ "phirehose - Project Hosting on Google Code") libs (included). In future, when the Twitter Site Streams API is out of beta, Site Streams will be supported by this plugin as well.

In order to absorb spikes in the data stream, the plugin uses a 'message queue' to decouple the process(es) that pull items off the stream(s) from the process that parses the items and adds information to the ThinkUp database.  
The plugin supports two different implementations of this message queue.  The first uses a new database table (`tu_stream_data`).  The second uses the [Redis](http://redis.io/ "Redis") persistent key-value store to support the message queue. (Gory details supplied on request, especially if you are interested in Redis' capabilities.)  

Redis is lightweight, robust, and efficient, and reduces database load; and so its use should be preferable in general to the db-based implementation.  However, the plugin can only use the Redis implementation if you are running a version of PHP >= 5.3. (Details: this is due to the Redis client libs we're using, [Predis](https://github.com/nrk/predis); and in particular the specific version of those libs that we are using.  If you really want to use Redis with PHP 5.2, it is possible with a bit of extra code modification.)

### Alpha Testing ###

For alpha testing, it will be useful to test this plugin using a variety of platforms and configurations.  It's been primarily tested thus far on OS X with PHP 5.3.3, and there should be no issues running on Linux.  We are looking for testers who run PHP 5.2 (to check that the plugin will properly fall back to using the db-based message queue instead of Redis, regardless of configuration settings).  It will be 'interesting' to see how the plugin runs on Windows+Cygwin. (On Windows, Cygwin will be required). It would be helpful also to have the Redis integration tested on different platforms.

## Setup and Config ##

Use this branch: [https://github.com/amygdala/thinktank/tree/streaming4](https://github.com/amygdala/thinktank/tree/streaming4).
You can grab it by clicking the 'Download' button on the page.
<br><strong>Note</strong>: this link was updated on 2011-02-24 to point to the `streaming4` branch.

    
Then do the following:    

- Optional: install Redis, if you are willing to test that aspect of the plugin: [http://redis.io/](http://redis.io/).  It would be useful to us if you are willing to try it. You may need to compile it, which should be straightforward.

- back up/copy your database.  [There are no known database issues with this code, but do this to be on the safe side.]

- run the db migration `<thinkup>/webapp/install/sql/mysql_migrations/2011-03-19_streaming.sql`. [This assumes that your copy of the database is already otherwise current-- if you are starting with an older version of the ThinkUp database you may first need to run additional ThinkUp migrations as you would normally do to update to a new release of the master, as described in the ThinkUp documentation.]

  **Update 2011-03-19**: The [streaming4](https://github.com/amygdala/thinktank/tree/streaming4) branch has now been rebased on top of beta 9.  This means that if you update on top of an earlier installation, you'll need to run the beta 9 schema migration (and update the `database_version` in the `tu_options` table ). There were no new changes to the streaming branch's schema migration, but its migration file, `2011-03-19_streaming.sql`, has been renamed so that it is dated after the beta 9 migration file.

  **Updating the database from a previous version of the streaming branch**: There have been a few schema changes from previous streaming versions, notably a new table and a new table field, as well as a couple of index improvements.  If you want to update from a previous installation of the streaming branch, the most straightforward approach is probably to first run the migration file above (ignoring any complaints from the script about adding already-existing fields). Then [make this set of changes](https://gist.github.com/cffeba6df6043c328d42) to the existing tables.


- when you go to 'Settings' in your ThinkUp installation in the browser, you should now see a "**Twitter Realtime**" plugin.  Activate it, then go into its configuration page.  For the Twitter Realtime plugin to work properly, the Twitter Plugin must already be activated and configured with a Twitter app 'consumer key' and 'consumer secret'.  Additionally, you should have already set up one or more Twitter accounts via the Twitter Plugin.

- in the Twitter Realtime plugin config, enter the path to your php interpreter (e.g. `/usr/bin/php`). This is required.  Double check that you've typed it correctly.

- to use Redis, go into the advanced plugin options and enter 'true'.  If that is set, the plugin will expect to find a Redis server running (as described below). Otherwise, if it is left blank or set to something other than 'true', the db-based queue will be used. Also, as discussed above, if you are not running PHP >= 5.3, then the database queue will be used even if you configure the plugin to use Redis.

- The Twitter Realtime plugin should list all the Twitter accounts (instances) in your ThinkUp installation. This is the same list you see in the Twitter Plugin config page. Pause/start the Twitter instances as you prefer.  Any instances that you pause will be paused in the crawler as well. 
A Twitter User Stream will be opened for each active (unpaused) Twitter account in separately-running processes, up to 5 accounts max (to avoid running afoul of Twitter guidelines).  

- set `slog_location` in the `config.inc.php` file to point to where a new streaming log will be, e.g.:

    `$THINKUP_CFG['slog_location']   = $THINKUP_CFG['source_root_path'].'logs/streaming.log';`

  Then, you will also need to create that file, `streaming.log`. E.g., 

    <code><pre>
    % cd &lt;thinkup&gt;/logs
    % touch streaming.log
    </pre></code>

## Running the scripts ##

If you are using Redis, first start up the Redis server in its own terminal window if you want to use it: 
cd to `<redis_installation>/src` and start the server:

    % ./redis-server
    
Note: once started, the Redis server will run happily in the background after its window is closed.  Similarly, some Redis installations will by default start the server automatically.  So, if you get a diagnostic that the redis server port is already in use, check for already-running processes.

To start up the instance streams, in another terminal window, cd to `<thinkup>/webapp/crawler`.  Then do:

    % php stream.php stream <admin_user_email> <pwd>
    
(E.g., use the same admin email and pwd that you use to run the crawler). You should see some output to STDOUT along the lines of the following, which in this example shows starting up three instances.  In this example, IDs 1 and 3 belong to one user, and ID 2 belongs to a different user. 

      have streamer method: stream
      in TwitterRealtimePlugin->stream_all()
      starting new process for amy@infosleuth.net and 1
      started pid 42486 for amy@infosleuth.net and instance id 1
      starting new process for amy@infosleuth.net and 3
      started pid 42488 for amy@infosleuth.net and instance id 3
      starting new process for info+bob@infosleuth.net and 2
      started pid 42490 for info+bob@infosleuth.net and instance id 2


The startup script will then exit.  A child script, `<thinkup>/webapp/plugins/twitter_realtime/streaming/stream2.php`, is used to launch the individual instance streams.  These are the processes which write to the message queue.  
[`shell_exec` is used to launch the child processes-- please let me know if this acts up for anyone].

At this point you might check that all the intended streams are running, via:

        ps auxw | grep -i stream2 

You should see a process for each instance.  You will also find some temp log files, for the output of each of the above processes, in the `<thinkup>/logs` dir. They will have the format `<user_email>_<inst_id>.log`.  If there were to be any trouble opening up the individual streams, these logs are where the problems would be reported. (Ensure that the login under which you're running these scripts can write to your `logs` directory).


Next, start up the stream processor. This is the process that reads from the message queue and processes the data. In another terminal window, again cd to `<thinkup>/webapp/crawler`.
Then run:  

        % php stream.php stream_process <admin_user_email> <pwd>
        
Once this process is running, you will see output generated in the `<thinkup>/logs/streaming.log` file (or whatever streaming log location you specified in the config file).
  
Once all the scripts are up and running, you can see the new realtime content displayed right away in the web app.

There is no problem in running the crawler at the same time as the streaming scripts are running. (One thing the crawler will do is expand the URLs collected by the streaming processes, if you have the `Expand URLs` plugin activated).
  
        
### Shutting down the streams ###

To shut down the stream handling processes for the Twitter instances, do from `<thinkup>webapp/crawler`:

    % php stream.php shutdown_streams <admin_user_email> <pwd>
    
You should see some output along these lines:

      have streamer method: shutdown_streams
      in TwitterRealtimePlugin->shutdown_streams()
      killing all running streaming processes
      killing: 42486
      killed: 42486
      killing: 42488
      killed: 42488
      killing: 42490
      killed: 42490

    

To shut down the 'stream processor' script (the one you started via `php stream.php stream_process`), just ctl-C in its terminal window.   
You can use ctl-C to shut down the Redis server also.

### Automatically Restarting Streams as Necessary ###

If you should run `php stream.php stream <admin_user_email> <pwd>` while stream processes are already opened, it will check to see which streams show signs of recent activity, where 'recent' is currently defined to be 10 minutes. [Note: this should perhaps be a user-configurable value.].  Those streams showing activity will not be restarted. Those streams that appear inactive will be killed and restarted.  So, you can run this command regularly without doing any harm.  
  
If all of the streams in the example above were still running, `php stream.php stream <admin_user_email> <pwd>` would generate output like this:

      have streamer method: stream
      in TwitterRealtimePlugin->stream_all()
      process 52389 listed with recent update time for instance with amy@infosleuth.net and 1-- not starting another one
      process 52828 listed with recent update time for instance with amy@infosleuth.net and 3-- not starting another one
      process 52393 listed with recent update time for instance with info+bob@infosleuth.net and 2-- not starting another one
      
If one of the streams had died for some reason, you would instead see output along these lines:

      have streamer method: stream
      in TwitterRealtimePlugin->stream_all()
      process 52389 listed with recent update time for instance with amy@infosleuth.net and 1-- not starting another one
      killing process 52828 -- it has not updated recently
      sh: line 0: kill: (52828) - No such process
      starting new process for amy@infosleuth.net and 3
      started pid 53785 for amy@infosleuth.net and instance id 3
      process 52393 listed with recent update time for instance with info+bob@infosleuth.net and 2-- not starting another one


</body>
</html>
