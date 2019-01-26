## Table of contents

* [Description](#description)
* [Author. License](#author-license)
* [Tabs and Spaces](#tabs-and-spaces)
* [Pipe](#pipe)
* [Variable names](#variable-names)
* [Function names](#function-names)
* [Error handling](#error-handling)
* [Pipe error handling](#pipe-error-handling)
* [Automatic error handling](#automatic-error-handling)
  * [Set -u](#set--u)
  * [Set -e](#set--e)
  * [No excuse](#no-excuse)
* [Catch up with $?](#catch-up-with-)
* [Shell or Python/Ruby/etc](#shell-or-pythonrubyetc)
* [Good lessons](#good-lessons)
* [Resources](#resources)

## Description

This is a set of `Bash` coding conventions and good practices.

## Authors. License

The original author is Anh K. Huynh and the original work was part of
[`TheSLinux`](http://theslinux.org/doc/bash/coding_style/).

A few contributors have been helped to fix errors and improve the style.
They are also the authors.

The work is released under a MIT license.

## Tabs and Spaces

Never use `(smart-)`tabs. Replace a tab by two spaces.

Do not accept any trailing spaces.

## Pipe

There are `inline` pipe and `display` pipe.  Unless your pipe is too
short, please use `display` pipe to make things clear.

Example

    This is an inline pipe: "$(ls -la /foo/ | grep /bar/)"

    # The following pipe is of display form: every command is on
    # its own line.

    _foobar="$( \
      ls -la /foo/ \
      | grep /bar/ \
      | awk '{print $NF}')"

    _generate_long_lists \
    | while IFS= read -r  _line; do
        _do_something_fun
      done

When using `display` form, put pipe symbol (`|`) at the beginning of
of its statement. Never put `|` at the end of a line.

## Variable names

A variable is named according to its scope.

* If a variable can be changed from its parent environment,
  it should be in uppercase; e.g, `THIS_IS_A_USER_VARIABLE`.
* Other variables are in lowercase, started by an underscore;
  e.g, `_this_is_a_variable`. The primary purpose of the underscore (`_`)
  is to create a natural distance between the dollar (`$`)
  and the name when the variable is used (e.g, `$_this_is_a_variable`).
  This makes your code more readable, esp. when there isn't color support
  on your source code viewer.
* Any local variables inside a function definition should be
  declared with a `local` statement.

Example

    # The following variable can be provided by user at run time.
    D_ROOT="${D_ROOT:-}"

    # All variables inside `_my_def` are declared with `local` statement.
    _my_def() {
      local _d_tmp="/tmp/"
      local _f_a=
      local _f_b=

      # This is good, but it's quite a mess
      local _f_x= _f_y=
    }

Though `local` statement can declare multiple variables, that way
makes your code unreadable. Put each `local` statement on its own line.

`FIXME`: Add flexibility support.

## Function names

Name of internal functions should be started by an underscore (`_`).
Use underscore (`_`) to glue verbs and nouns. Don't use camel form
(`ThisIsBad`; use `this_is_not_bad` instead.)

## Error handling

All errors should be sent to `STDERR`. Never send any error/warning message
to a`STDOUT` device. Never use `echo` directly to print your message;
use a wrapper instead (`warn`, `err`, `die`,...). For example,

    _warn() {
      echo >&2 ":: $*"
    }

    _die() {
      echo >&2 ":: $*"
      exit 1
    }

Do not handle error of another function. Each function should handle
error and/or error message by their own implementation, inside its own
definition.

    _my_def() {
      _foobar_call

      if [[ $? -ge 1 ]]; then
        echo >&2 "_foobar_call has some error"
        _error "_foobar_call has some error"
        return 1
      fi
    }

In the above example, `_my_def` is trying to handle error for `_foobar_call`.
That's not a good idea. Use the following code instead

    _foobar_call() {
      # do something

      if [[ $? -ge 1 ]]; then
        _error "$FUNCNAME has some internal error"
      fi
    }

    _my_def() {
      _foobar_call || return 1
    }

## Pipe error handling

Pipe stores its components' return codes in the `PIPESTATUS` array.
This variable can be used only *ONCE* in the sub-`{shell,process}`
followed the pipe. Be sure you catch it up!

    echo test | fail_command | something_else
    local _ret_pipe=( "${PIPESTATUS[@]}" )
    # from here, `PIPESTATUS` is not available anymore

When this `_ret_pipe` array contains something other than zero,
you should check if some pipe component has failed. For example,

    # Note:
    #   This function only works when it is invoked
    #   immediately after a pipe statement.
    _is_good_pipe() {
      echo "${PIPESTATUS[@]}" | grep -qE "^[0 ]+$"
    }

    _do_something | _do_something_else | _do_anything
    _is_good_pipe \
    || {
      echo >&2 ":: Unable to do something"
    }

## Automatic error handling

### Set -u

Always use `set -u` to make sure you won't use any undeclared variable.
This saves you from a lot of headaches and critical bugs.

Because `set -u` can't help when a variable is declared and set to empty
value, don't trust it twice.

### Set -e

Use `set -e` if your script is being used for your own business.

Be **careful** when shipping `set -e` script to the world. It can simply
break a lot of games. And sometimes you will shoot yourself in the foot.

Let's see

    set -e
    _do_some_critical_check

    if [[ $? -ge 1 ]]; then
      echo "Oh, you will never see this line"
    fi

If `_do_some_critical_check` fails, the script just exits and the following
code is just skipped without any notice. Too bad, right?

Another example, in effect of `set -e`:

    (false && true); echo not here

print nothing, while:

    { false && true; }; echo here

print `here`.

The result is varied with different shells or even different versions of the same shell.

In general, don't rely on `set -e` and do proper error handling instead.

For more details about `set -e`, please read

* http://mywiki.wooledge.org/BashFAQ/105/Answers
* [When Bash scripts bite](https://news.ycombinator.com/item?id=14321213)

### No excuse

When someone tells you to do something, you may blindly do as said,
or you would think twice then raise your white flag.

Similarly, you should give your script a white flag. A backup script
can't be executed on any workstation. A clean up job can't silently
send `rm` commands in any directory. Critical mission script should

* exit immediately without doing anything if argument list is empty;
* exit if basic constraints are not established.

Keep this in mind. Always.

## Catch up with $?

`$?` is used to get the return code of the *last statement*.
To use it, please make sure you are not too late. The best way is to
save the variable to a local variable. For example,

    _do_something_critical
    local _ret="$?"
    # from now, $? is zero, because the latest statement (assignment)
    # (always) returns zero.

    _do_something_terrible
    echo "done"
    if [[ $? -ge 1 ]]; then
      # Bash will never reach here. Because "echo" has returned zero.
    fi

`$?` is very useful. But don't trust it.

## Good lessons

See also in `LESSONS.md` (https://github.com/icy/bash-coding-style/blob/master/LESSONS.md).

## Shell or Python/Ruby/etc

In many situations you may have to answer to yourself whether you have
to use `Bash` and/or `Ruby/Python/Go/etc`.

One significant factor is that `Bash` doesn't have a good memory.
That means if you have a bunch of data (in any format) you probably
reload them every time you want to extract some portion from them.
This really makes your script slow and buggy. When your script
needs to interpret any kind of data, it's a good idea to move forward
and rewrite the script in `Ruby/Python/Golang`.

Anyway, probably you can't deny to ignore `Bash`:
it's still very popular and many services are woken up by some shell things.
Keep learning some basic things and you will never have to say sorry.

## Resources

* [Google - Shell Style Guide](https://google.github.io/styleguide/shell.xml)
* [Defensive Bash programming](https://news.ycombinator.com/item?id=7815190)
* [Shellcheck](https://github.com/koalaman/shellcheck)
