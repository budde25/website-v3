+++
title = 'CLI Tips - Paging'
date = 2024-06-28T10:14:38-05:00
draft = true
+++

During your development of your Command Line Interface (CLI) you may have encountered a moment where output a enormous amount of text.
While this may be fine design you may also want to conider automtaically paging the output for the user.

In this post, we'll explore the ins and out of paged output, discussing:

* The benefits of using a pager to streamline your terminal experience
* When to invoke a pager: tips for identifying situations where paged
output is most useful
* How to customize your pager settings to suit your needs

## Background

For context I first started experimenting with paging after developing a CLI tool whose primary purpose is displaying large log files.
Most of the output were hundreds if not thousands of lines long.
This meant that every time I ran the program it would just spit text to my terminal destroying my scrollback but also make the data I needed hard to find.
After a while found that I was unconditionally piping the output to less.

```console
my-awesome-cli | less
```

This got me thinking, should my program just invoke the the pager automatically?

## Prior art

Automatic invocation of a pager is nothing new.
Plenty of programs do it.
If you've ever used `git log` or `git diff` you've probably noticed this yourself.

## Should I Page Output?

This one is pretty simple imo, if you are regularly printing out more that a screens worth of data you should consider paging the output.
Don't go overboard, but even a sufficiently complicated CLI tool might want to consider paging their `--help` output. Ex: `fish --help`.
I for once don't like to fiddle around my terminal trying to search for a part of the output only to stubble upon stale data from a previous invocation.

## CLI option

I would suggest adding an option to you CLI either in the format `--no-pager` to disable the pager or `--pager <PAGER>` option where PAGER can be 'system', 'none' or another value such as 'builtin'.
The advantage of the latter is that you can compile/ship a pager in your program to not rely on the system having one.

## Choosing a Pager

Right away the obvious thing to do is just to spawn `less` piping you output to the child process.
This is a good start however it leaves a lot to be desired.

* What if the user doesn't have `less` installed?
* How do I handle the `PAGER` environment variable?
* How do I deal with someone not wanting to page output?

The ideal order is to check the CLI options to see if the pager has been explicitly disabled. If so print to stdout.
Then once you decided you would like to page the output, start with the `PAGER` variable.
The `PAGER` may be unset or it can contain anything that would be passed into `sh -c` so don't expect just a path to a program.

You may want to consider a new variable for you program so that a user can change just you programs pager such as `$<MY CLI HERE>PAGER`.
Any failure spawning `less` or the `PAGER` can be handled by just print to stdout like normal.

## `less` Options

There are a few good to consider options when invoking the `less` pager:

* `-F or --quit-if-one-screen`
    > Causes less to automatically exit if the entire file can be displayed on the first screen.

    This is a very convenient option because it allows you to unconditionally pipe all your output to `less`.
    If its less than a screens worth it will just automatically print to stdout as if `less` was never invoked in the first place.
* `-R or --RAW-CONTROL-CHARS`
    This option lets the color and OCS 8 hyperlink escape characters to output in 'raw' form allowing them to work within `less`.
* `-X or --no-init`
    > Disables sending the termcap initialization and deinitialization strings to the terminal.

    This option stops less from clearing the screen when it closes, in effect all the lines on screen will remain in the terminal after quitting, but not the entire output.

As always refer to the `less` man page for more, its quite extensive.

## Using `less` options

As stated above it would be a good idea to respect the user choice and default to their preferred pager, however the options above may not be compatible for with their pager of choice.
For this reason you should execute their `PAGER` as is with any option and/or it may or may not have.
Consider only adding the `-FRX` if you default to the `less` as the alternative is need to parse the `PAGER` variable.

This leads to the preferred option, if you always want to execute with those options, pass it in the environment.
The `LESS` environment variable can be set with the flags. For example:

```console
LESS="FRX"
```

Another `less` environment variables to consider is `LESSCHARSET`. For example:

```console
LESSCHARSET="UTF-8"
```

Simply pass these to the child process to get all the configuration you could want. As always refer to the man pages for more!

## Bonus: `less` custom prompt

A lesser known feature of `less` is that it supports custom prompts.
This is useful if you want to provide context for your data.

This can be passed as a CLI option using `-P` or appended to the `LESS` variable with your other options.

```console
LESS="Ps<custom prompt>$"
```

One important note is that you must escape following characters: `'?', ':', '.', '%', '\'`. Refer to the less man page for more info.

Example escaping in Rust:

```rust
fn less_escape(string: &str) -> Cow<str> {
    if !string.contains(['?', ':', '.', '%', '\\']) {
        return Cow::Borrowed(string);
    }

    let mut string_escaped = String::with_capacity(string.len() + 1); // at least the len + 1
    for char in string.chars() {
        // According to less man page the following need to be escaped
        // question mark, colon, period, percent, and backslash
        if matches!(char, '?' | ':' | '.' | '%' | '\\') {
            string_escaped.push('\\');
        }
        string_escaped.push(char);
    }
    Cow::Owned(string_escaped)
}
```

## Builtin Pager

One option to make you CLI more portable is shipping with a custom pager.
Some examples I know of are [pypager](https://github.com/prompt-toolkit/pypager) and [minus](https://github.com/AMythicDev/minus)[^1].
These are not nearly as feature complete as `less` and with `less` being so prevalent I would recommend only shipping these as an option and not necessarily defaulting to them.

[^1]: I have never used either of these libraries so I cannot endorse them. But they both seem like interesting projects.

## Trivia 😄

Before `less` it was common to use the `more` pager.
It has since been replaced by `less` pager and in fact invoking `more` is usually just a symlink (or hardlink) to `less`.
Output on macos:

```console
$ more
Missing filename ("less --help" for help)
```
