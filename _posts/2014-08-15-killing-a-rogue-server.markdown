---
layout: post
title:  "Killing a rogue server to free up its port"
---
**TL;DR**:
run the following, replacing PORT_NUMBER with the blocked port, and PID with the process ID of the process that lsof indicates is using the port:

{% highlight bash %}
lsof -i :PORT_NUMBER
sudo kill PID
{% endhighlight %}

Wait a few moments, run `ps aux | grep PID`. if the process still comes up, then `sudo kill -9 PID`.

## Long explanation: Process already using a port and you want to know what to do? Read on.
It's common to see students in my class at [MakerSquare SF](http://www.makersquare.com) struggle when they try to start up a ruby webrick server or a node server and find out some other [process](http://en.wikipedia.org/wiki/Process_(computing)) is already using the default [port]((http://en.wikipedia.org/wiki/Port_(computer_networking)) of their server.  Sometimes this is because a previous server didn't terminate properly, sometimes it's because they have a terminal window minimized somewhere running a server, sometimes its because they are attempting to use a port locally that is currently being forwarded to a vagrant VM. Regardless of the cause, there is a really easy way to both figure out what the heck is going on and terminate the process currently using the port.

## Step 1: Figure out what is running on the port using lsof
`lsof` is a command available on most Unix-like operating systems (like OSX, or any flavor of Linux/xBSD) that is used to list currently open files, but we can also use it to list all the processes that are using a given port number using the `-i` option. We can use the `-i` flag to specify only the open files listening on a given Internet address (remember, in Unix, [everything is a file](http://en.wikipedia.org/wiki/Everything_is_a_file), including running processes like web servers). By specifying the port you want to use using the syntax `lsof -i :PORT_NUMBER`, we can list all the servers running on a given port.

Try this command out: `lsof -i :8080`. If you aren't running an application that is currently using port 8080, nothing will return and we can try running an application on this port. (if something is running on the port already, try another number like 8001 or 8808, it really doesn't matter which one in particular for this example).

Start a python simpleHTTPServer on port 8080 with the following command: `python -m SimpleHTTPServer 8080`. If you are not using `8080`, replace it with whatever port number you chose before.

(Note: you will need some version of Python installed for this command, most linux distros and OSX have Python installed by default, but if your OS doesn't, you'll need to google for instructions on how to install that for your OS.)

Now, run `lsof -i :8080` again, and it should produce output similar to the following:

{% highlight bash %}
CMD     PID   USER         NAME
Python  29203 my_username  TCP *:http-alt (LISTEN)
{% endhighlight %}

(Note: I edited out some of the columns to make the output fit this blog better.)

This response tells us that there is currently one process using port 8080, that the command running it is `Python`, that it has a [Process ID](http://en.wikipedia.org/wiki/Process_identifier) of `29203`, started by `my_username`. If this was a rogue process, we could decide at this point whether or not this was a process we intended to run on port 8080, in which case we'd probably want to leave it alone, or if it was a process that should be terminated, in which case we can use the PID to terminate the process.

## Step 2: Terminating the process using kill
Yes, if you don't know, there is totally a `kill` command on Unix systems. Confusingly, it's actually an application to send various signals to your processes. Typically the signals you send to processes with `kill` are to ask the process to terminate itself (though there are other signals you can send that have nothing to do with terminating the process), hence the name. We can use signals, combined with the PID gathered from using `lsof` in the previous step, to terminate a process that is using a port that it shouldn't be using.

There are something like 30+ different [Unix signals](http://en.wikipedia.org/wiki/Unix_signal) we can send but we're going to only look at a few today: the terminate signal (SIGTERM), and the kill signal (SIGKILL). Both of these signals are used for generally the same purpose: terminating applications. The difference comes in how they actually terminate.

SIGTERM is a general terminate signal, it's a way of asking an process nicely to terminate. This will give the process  If the application isn't programmed to respond to SIGTERM by quitting, it might just ignore it, or if the application is stuck in a bad state, it might also not quit if you run a SIGTERM. There are a few other signals that operate similarly to SIGTERM that you can read about: SIGINT, which is the signal sent when you `ctrl+c` a process to terminate it, and SIGHUP, which is a signal to indicate a process has been disconnected from the terminal in particular are useful to read about.

SIGKILL, unlike SIGTERM, does not ask nicely before terminating a process. It is the equivalent of ripping the power cord out of a socket to turn something off. Unlike the other signals, it doesn't ask the process to terminate, it just executes without remorse. This can be risky if the process is currently writing to disk or hasn't finished using resources, so for this reason, I usually precede any SIGKILL with SIGTERM or its equivalent signals shown above since it is almost always preferable to have a process terminate itself than to force it to terminate. That said, you'll still find plenty of occasions where a SIGKILL is the only thing that will terminate a process, especially when a process has hanged or is completely malfunctioning.

To send these signals, you can use the `kill` command along with their signal name and the offending process's PID, so for example, if I still had that Python process running from Step 1 on PID 29203, I could use `sudo kill -SIGTERM 29203` to send a SIGTERM to the process. After waiting a few moments for the process to terminate itself, I could then either use `lsof` again to see if the port had been freed, or use `ps aux | grep 29203` to see if the process is still currently running. If it is, I could then send a SIGKILL if the process didn't terminate as expected.

There is also a shorthand for each of these signals. Each signal has a [numerical identifier](http://people.cs.pitt.edu/~alanjawi/cs449/code/shell/UnixSignals.htm), for SIGTERM and SIGKILL they are 15 and 9 respectively. Instead of using the full name with kill, you can use just the signal numbers to send a terminate signal. E.g. `sudo kill -9 29203` to send a SIGKILL to that python process before. With SIGTERM you don't even need to pass in the signal, just using `sudo kill 29203` is sufficient to send a SIGTERM.

Phew. That's a lot to go through for a simple concept, but hopefully now you understand a little better about how to 1. figure out what processes are currently using a port on your machine, and 2. how to terminate a given process once we have a PID.
