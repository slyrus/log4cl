# Quickstart
## Installation and loading

The easiest way to install LOG4CL is to use
[QuickLisp](http://www.quicklisp.org) local-projects feature.

    $ cd ~/quicklisp/local-projects
    $ git clone git://github.com/7max/log4cl.git 

Then use the `(ql:quickload :log4cl)` from `REPL` to load it. Do
not try to include LOG4CL into your package's *:use* list. LOG4CL package is
designed to be used by referencing its symbols with the package prefix.

## The Basics

     (log:info "Hello World")

         [14:39:03] [info] <cl-user> - Hello World

     (log:i "I'm really lazy")

         [14:39:08] [info] <cl-user> - I'm really lazy

     (log:debug "something")

As you can see package contains one letter shortcuts for all the log
levels, for the lazy amongst us. Also note that debug log statement
did not produce any output. Changing the log level is done like this:

    (log:config :debug)

    (log:debug "something")

    [14:41:44] [debug] <cl-user> - something

As with the log statements, both the log level and the `config`
function can also be abbreviated to just a letter, for
example one could have typed `(log:c :d`) to change log level
to `debug`

One additional logging utility function `(log:expr)`, helps you to
quickly debug values of variables and expressions:

    (let ((a 1) (b 2))
      (log:expr a b (+ a b)))

    [14:54:39] [debug] <cl-user> - A=1 B=2 (+ A B)=3

This function also has two aliases: `(log:sexp)` and `(log:s)`

### Log severity levels 
LOG4CL has all the same log levels that Log4J does, and some
additional ones. They are in the order of decreasing severity:
*fatal*, *error*, *warn*, *info*, *debug*, *user1* through *user4*,
*trace*, and finally *user5* through *user9*

There is also two special log levels *off* and *unset*, which are used
to turn the logging off, and to inherit the log level from the parent
logger. The default log level for all loggers is *unset*.

When logging is enabled for a level, it is also enabled for levels
with more severity. For example if *debug* level is enabled, then
log messages with *info* and *error* level will also be enabled.

### How to determine of logging is enabled?
-----------------------------------------------------------------
Sometimes its useful to figure out if logging is enabled for a particular
level, to conditionally execute log statements. The logging function
without any arguments, will return `T` or `NIL` depending if logging
on that level is enabled.

    (when (log:debug)
      (print-detailed-debugging-info))

## Loggers and categories

You may have noticed that the log output in addition to the time of
the log message, its severity level and the actual log message,
contains the string `<cl-user>`, which corresponds to the current
package.

That string is called a category of the log message, and can also be
referred to as *logger's category*, or simply a logger.

*Logger* is an object, that acts as source of the log message, and
also as a configuration point, which can have its own log level and
appenders. Log messages are always logged into a specific logger
object.

Loggers form a hierarchy, starting from the root logger. Each logger
can have child loggers, and these loggers can have their own child loggers and
so on. The full category name of the logger, is concatenation
of all the parent logger names starting from the root logger, with some
separator string, which defaults to `":"`

So for a logger `A` which has a child logger `B`, which in turn has
child logger `C`, the full category name of the logger C is `A:B:C`

### Automatic naming of loggers

When you issue a log statement such as `(log:info "whatever")` LOG4CL
determines the logger name automatically based on surrounding content.
That's why, when you issue log statement from `REPL` with the current
package being *CL-USER*, the logger message is logged into is also
called `CL-USER`.

You can directly instantiate logger objects, by using
the `(log:make-logger)` function:

    (log:make-logger)

    #<LOGGER CL-USER>

    (log:make-logger :a)

    #<LOGGER CL-USER:A>

    (log:make-logger '(one two three))

    #<LOGGER ONE:TWO:THREE>

In the first example, the logger name is automatically determined from
context, just as with the default log statement. In the second
example, specifying a keyword as the logger name, will create a
sub-logger of such default logger. And finally the most generic syntax
is using a quoted list, where each element names a logger starting
from root.

When submitting a log message, you can specify a logger object as a
first argument:

    (log:info (log:make-logger :a) "goes to logger A")

    [15:14:51] [info] <cl-user:a> - goes to logger A

    (log:info (log:make-logger '(one two three)) "goes to logger ONE:TWO:THREE")

    [15:15:04] [info] <one:two:three> - goes to logger ONE:TWO:THREE


As a shortcut you can omit the `(log:make-logger`) and specify a
`(make-logger)` argument directly as a first argument of a logging
function, with exception of `(log:expr)`, with which you have to
always use explicit call to `(log:make-logger)`.

    (log:info '(one two three) "also goes to logger ONE:TWO:THREE")

    [15:18:02] [info] <one:two:three> - also goes to logger ONE:TWO:THREE

    (log:info :b "also goes to logger CL-USER:B")

    [15:18:15] [info] <cl-user:b> - also goes to logger CL-USER:B

### Automatic naming inside a function

Under SBCL automatic logger naming goes farther then just naming
the logger after the current package:

    (defun foo (x)
      (log:expr x (1+ x))) 

    (foo 3)

    [16:47:09] [debug] <cl-user:foo> - X=3 (1+ X)=4

    (defun foo (x)
      (flet ((baz ()
               (log:expr x (1+ x))))
        (baz)))

    (foo 3)

    [16:47:09] [debug] <cl-user:foo:baz> - X=3 (1+ X)=4

It also works with methods:

    (defmethod bar ((x number) y)
      (flet ((baz ()
        (log:expr x (1+ x) y)))
           (baz)))

    (bar 3 'foo)

    [16:52:15] [debug] <cl-user:bar:number:baz> - X=3 (1+ X)=4 Y=FOO

    (defmethod bar :before (x y) (log:info "what happens before ~s ~s" x y))

    (bar 41 t)

  
    [16:55:11] [info] <cl-user:bar:before> - what happens before 41 T
    [16:55:11] [debug] <cl-user:bar:number:baz> - X=41 (1+ X)=42 Y=T

### Configuring a specific logger category

By default a call to `(log:config`) function configures the root
logger. All the rest of the loggers in the system inherit the log
level from their parent loggers, and eventually from the root logger.

It is possible to configure each logger to have its own log level, which
will override the one inherited from the parent:

    (log-config '(cl-user a) :debug)
    (log-config '(cl-user b) :trace)
    (log-config '(cl-user) :info)

    (log:info "this is info")

    [15:26:50] [info] <cl-user> - this is info

    (log:debug "this is debug")

    (log:debug :a "this is debug")

    [15:27:05] [debug] <cl-user:a> - this is debug

    (log:trace :a "this is trace")

    (log:trace :b "this is trace")

    [15:27:22] [trace] <cl-user:b> - this is trace   
    
In the above example we had configured logger `CL-USER` `CL-USER:A`
and `CL-USER:B` with the *info*, *debug* and *trace* log levels, and
tested submitting log messages to them. To make logger inherit its log
level from its parent again, use the log level *unset* or use the
*:clear* option of the configuration function.

    (log:config '(cl-user) :unset)

    (log:debug "test")

    [15:34:37] [debug] <cl-user> - test

    (log:config '(cl-user) :info :clear)

    (log:debug :a "this is a trace")

Note: *:clear* option resets the log level of all the child loggers of
the specified logger, but not the logger itself. To do both use
`(log:config '(logger) :unset :clear)`

### Displaying logger hierarchy configuration

Without any arguments, `(log:config)` will display logging hierarchy
and its configuration on the standard output. Only "interesting"
loggers will be displayed, "interesting" defined as having either an
explicit log level, or having any appenders, or being non-additive
(meaning they don't propagate messages to the ancestor's appenders)

    CL-USER> (log:config)
    ROOT, DEBUG
    |
    +-#<CONSOLE-APPENDER {102576AD21}>
    |     with #<PATTERN-LAYOUT {102566A5E1}>
    |          :conversion-pattern "[%D{%H:%M:%S}] [%P] <%c{}{}{:downcase}> - %m%n"
    |     :immediate-flush NIL
    |     :flush-interval  1
    |
    +-LOG4CL-IMPL
      |
      +-SELF (non-additive), WARN
        |
        +-#<CONSOLE-APPENDER {101EAEB6B1}>
              with #<PATTERN-LAYOUT {101E55FC21}>
                   :conversion-pattern "[%d{%H:%M:%S}] [%-5P] <%c{}{}{:downcase}>%n  *%I{>} %m%n"
              :immediate-flush NIL
              :flush-interval  1
    CL-USER> 

## Appenders and layouts

While loggers are logical sinks for the log messages, appenders are
physical ones, that are actually responsible formatting and routing
the formatter log message somewhere.

For example `CONSOLE-APPENDER` writes message to `*DEBUG-IO*` stream
and `DAILY-FILE-APPENDER` writes messages into a file with
configurable file-name.

### Layouts

Appenders format the log messages by means of layouts. Layout receives
information about logging event, and the stream to print output into,
and is responsible for actual formatting of the message, such as
adding timestamps or other formatting.

LOG4CL provides two layout classes:

- `SIMPLE-LAYOUT` formats the message in a very primitive way, by
  printing the log level followed by a dash and the user log message

- `PATTERN-LAYOUT` formats the message by specifying a conversion
  pattern string, which has escape sequences for various part of the
  log message. See Lisp docstring for `LOG4CL:PATTERN-LAYOUT` class
  for the description of the pattern layout elements.

### Configuring appenders and layouts with `(log:config)` function

The versatile `(log:config)` function can also add and remove
appenders and their layouts. See the Lisp docstring for full
description, below are a few examples:

- `:SANE` option removes all appenders from the logger, and adds a
  `LOG:CONSOLE-APPENDER` with a pattern layout used in throughout the
  examples in this document

            (log:config :sane)

            (log:info "test")

            [15:54:04] [info] <cl-user> - test

- `:TWOLINE` option changes the pattern layout used to have output on
  two lines, the header line with time-stamp, log level, and logger's
  category, and second line with the actual log message.

              (log:config :sane :twoline)

              (log:info "test")

              [15:56:14] [ info] <cl-user>
                * test


- `:DAILY` option adds a `LOG:DAILY-FILE-APPENDER` appender, logging
  into the specified file. The file will be switched at midnight, with
  the log renamed by adding `<year><mon><day>` suffix to its name.

              (log:config :daily "log.txt")

              (log:info "test")

              [15:56:14] [ info] <cl-user>
                * test

              $ cat log.txt
              [15:58:50] [info] <cl-user> - test
  It will use same pattern layout as the *:sane* option, it will also
  accept *:twoline*. If both *:sane* and *:daily* are used, then
  console appender is not added, you can use *:console* option to
  force adding both file and console appenders.

- `:PATTERN` option allows one to change the conversion pattern for
  the new pattern layout used with new appenders added by *:sane*,
  *:daily* and/or *:console* options

              (log:config :sane :pattern "%d -- %p -- %m%n")

              (log:info "test")

              2012-02-23 21:05:12 -- INFO -- test

