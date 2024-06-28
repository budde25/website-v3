+++
title = 'CLI Output Paging'
date = 2024-06-28T10:14:38-05:00
draft = true
+++

An important decision in CLI design is whether you should page your output. My initial reaction was: why would I? If the user wants to page let the pipe it to the pager themselves. I will walk you through my journey of deciding that for my CLI application.

For context, I was developering a CLI tool that primarily prints out large log files. Most of the files were hundreds if not thousands of lines long. This meant that everytime I ran the program it would just spit text to my terminal destroying my scrollback but also make the data I needed hard to find. I found that I was repetivly writing:

```sh
my-cli | less
```

to help navigate the output and save my terminal. This got me thinking, should my program just invoke the the pager automatically?

## Prior art

Automatic invocation of the Pager is nothing new, plenty of programs do it. If you've ever used `git log` or `git diff` you've probably noticed this yourself. After studying various implementations I have compiled what I learned.

## Should I Page Output?

This one is pretty simple imo, if you are regularly printing out more that a screens worth of data you should consider paging the output. Don't nessarily go overboard, but even a sufficiently complicated cli tool might want to consider paging their `--help` output

## CLI option

I would suggest adding an option to you CLI either in the format `--no-pager` to disable the pager or `--pager <PAGER>` where PAGER can be 'system', 'none' or another value such as 'builtin'. The advantage of the latter is that you can compile/ship a pager in your program to not rely on the system having one.

## System Pager `less`

Right away the obvious thing to do is just to spawn `less` piping you output to the stdin. This is a good start however it leaves a lot to be desired. What if the user doesn't have `less` installed? How do I handle the `PAGER` environment variable? How do I deal with someone not wanting to page output?

The correct order to handle a pager is to start with the `$PAGER` variable, then check the cli. You may want to consider a new variable for you program so that a user can change just you prgrams pager such as `<MY CLI>-PAGER`. Any failure such as `less` or the `$PAGER` not in `$PATH` can just print to stdout like normal.

### `less` options

There are a few good to consider options when invoking the `less` pager:

- `-F or --quit-if-one-screen`
    > Causes less to automatically exit if the entire file can be displayed on the first screen.

    This is a very convenient option since it means you can just pipe all output to less and if its less than a screens worith it will just automatically print as if less was never invoked in the first place.
- `-R or --RAW-CONTROL-CHARS`
    This option lets the color and OCS 8 hyperlink escape characters to output in 'raw' form allowing them to work within less.
- `-X or --no-init`
    > Disables sending the termcap initialization and deinitialization strings to the terminal.

    This option stops less from clearing the screen when it closes, in effect the final screen you were looking at will remain after quitting out which can be helpful.

## Picking a `$PAGER`

As stated above it would be a good idea to respect the user choice and default to their preferred pager, however the options obove may not be compatible for with thir pager of choice. Therefore a good approach would be to execute their `$PAGER` as is with any cli options it may or may not have. Only add the `-FRX` if you default to `less`.

Another option is to pass the options in the environment. This works great if you want the user to be able to let the user set their `$PAGER` to `less -i` and still want to use the above options without parsing their input. The `$LESS` environment variable can be set with the flags. For example:

```sh
LESS="FRX"
```

another less enviromment variables to consider `$LESSCHARSET`. For example:

```sh
LESSCHARSET="UTF-8"
```

## Bonus: `less` custom prompt

A lesser known feature of less is that you can pass a prompt in. This is useful if you want to provide context for your data.

This can be passed as a CLI option using `-P` or appended to the `$LESS` variable with your other options.

```sh
LESS="Ps<custom prompt>$"
```
One important not is the you must escape out the following characters: `'?', ':', '.', '%', '\'`. Refer to the less man page for more info.

Example escaping in rust:

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
