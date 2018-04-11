---
title: "SQL Commands for the Wise"
date: 2018-04-10T21:31:53-06:00
draft: true
---

If you're like me then you might have initially shyed away from using the `psql`
shell. It's nothing to be ashamed of, it's a bit intimidating and not the most
intuitive way to execute queries. Over time I've actually come to enjoy using 
it. And while I don't neccessarily find myself using it everyday, there are some
super handy commands I routinely use that I feel are worth passing on!

### `\h`
*help on syntax of SQL commands, * for all commands*

It took me a really long time ot find this command. It is incredibly helpful, 
especially when you consider how many times I forget the correct order of 
commands for `alter table` or how many times I've forgotten how to spell
`GRANT ALL ~PRIVELEGES~ PRIVILEGES`.

You can pass no arguments and get a list of all commands, or you can pass it
a specific command you need help with.

![](/img/help-copy.png)
*I'm sure you've never forgotten the syntax for `\COPY`...*

### `\x auto`
*toggle expanded output*

Another super handy one. My guess is that a lot of people already use this
but it's too good to risk not including in the list. `\x auto` makes reading
output from tables so much easier. Postgres will automagically toggle between
long/wide formats for you. I don't know how people lived before Postgres X.YY.

![](/img/x-auto.png)
*It even adjusts to your screen width!*

### `\timing`
*toggle timing of commands*

Great for benchmarking queries, `\timing` adds the number of seconds your 
query took to execute to the end of your results. Simple but oh so handy!


```
select avg(price), sum(1) from diamonds;
(1 row)
Time: 213.941 ms
```

### `\i`
*execute commands from file*

A lot of times I like to be able to interact with Postgres through the shell
but want to write and build my queries in an editor. This is pretty natural,
sometimes SQL queries can get quite lengthy! Stuffing that all into a prompt
is madness.

Enter the `\i <filename>` command. It allows you to specify a file where your
query is stored, and it will automatically execute that file in SQL.

![](/img/i-command.png)
*You think I made that histogram function all in the shell? No way José.*

### `\cd`
*change the current working directory*

Ever been in a Postgres session and realized that you're 5 directories
from the CSV you need to upload lives. It's super annoying and now, 100%
treatable with `\cd`. Just pass it the directory you'd like to move to
and `psql` will make it so.

It can work great in conjunction with the next command...


### `\!`
*execute command in shell or start interactive shell*

Are you one of those people who routinely types `ls` into your `psql` console
and feels like a total idiot--just hope nobody is looking over your shoulder. Happens
to me all the time! And while I might be playing up the embarrassment factor,
there's no need to play up just how handy it is to be able to run shell commands
from your Postgres session using the `\!` command. You can even "pipe" commands
to one another.

![](/img/bash-shell-pipe.png)
*Run commands straight from Postgres. Here I'm searching for a file, then piping to `head` to limit the number of files displayed.*

## Final Thoughts
- [Psql cheatsheet](http://www.postgresonline.com/downloads/special_feature/postgresql83_psql_cheatsheet.pdf)
- [Psql Guide](https://www.postgresql.org/docs/9.2/static/app-psql.html)
- [Talking to a PostgreSQL Data Base with psql](http://beige.ucs.indiana.edu/I590/node150.html)
- [Man Page!](http://linuxcommand.org/man_pages/psql1.html)
