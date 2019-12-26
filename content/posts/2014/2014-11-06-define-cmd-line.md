+++
title = "Define: Command Line"

[extra]
location = "San Francisco"
+++

If you didn't know, Google has a really useful [search
feature](http://www.google.com/intl/en/help/features.html) that allows you to
find the definition of any word or phrase when using the `define:` prefix on
your search term. This along with the built-in OSX dictionary provides an
incredibly useful set of tools to find any definition as quickly as possible.

<!-- more -->

However, if you're a terminal junkie like me, you spend a lot of time on the
command line. Sometimes you just need a definition at that instant and can't be
bothered to switch to another window to find it. Or perhaps you have a list of
words you'd like to quickly `grep` and find definitions. This is where
[sdvc](http://en.wikipedia.org/wiki/Sdcv) comes in.


## What is sdcv? And how do I get it?

sdcv is a console version of [StarDict](http://en.wikipedia.org/wiki/StarDict),
an open source utility used to access dictionary files in multiple languages
using a common dictionary file format.

The nature of the utility also allows you to download multiple dictionaries to
cull through. These dictionaries can range from English dictionaries to
dictionaries of hacker jargon, all searchable from the command line. Sound interesting? To install this utility, assuming you already have homebrew installed, all you need to do is:

    brew install sdcv

Unfortunately, the default installation doesn't come with any dictionaries. Go
ahead and choose the necessary
[dictionaries](http://abloz.com/huzheng/stardict- dic/dict.org/) you may need
or find useful. Dictionaries should be placed under `/usr/share/stardict/dic`
to ensure sdcv can detect it.


## Adding an alias for the define command

Now that you have sdcv installed, why use and remember an obscure name to query
your definitions. I added the following to my `.bash_profile` to define (hah) a
function that will call sdcv with some good defaults with whatever word or
phrase I pass to it.

``` bash
define () {
    # Check if sdcv is installed
    if command -v sdcv > /dev/null; then
        # -0 show UTF-8 output
        # -c show colorized output
        sdcv -0c "$@"
    else
        echo "You need to install sdcv."
    fi
}
```
