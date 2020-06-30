+++
title = "Adventures in Rust: CLI"

[taxonomies]
tags = ["rustlang"]

[extra]
location = "San Francisco"
+++

My second project in Rust is a little more practical than my first (which
you can read [here][rust-game]). This project involves creating a
command-line utility that is able to interact with the [Bear Writer][bear-app]
application, an OS X app that I use for taking notes, writing blogs, and
generally keeping my life organized in a single, cloud-synced place.

<!-- more -->
Bear is a truly wonderful app that lacks a CLI so when I’m in the terminal
it’s requires tediously switching back and forth between two screens.
Honestly, it’s not that big of a deal, but why not use it as an excuse to
build a little utility that I can use.

Plus I thought up a fantastic name for the utility, `cub` for
**C**ommand-line **U**tility for **B**ear.

You can check out the code for this [project here](https://github.com/a5huynh/cub-cli).

[bear-app]: https://bear-writer.com
[rust-game]: https://a5huynh.github.io/2018/02/02/adventures-in-rust.html


## What this will cover

Here are the things I cover in this post in case you want to skip around:
* Using `clap` to read values from the command-line.
* Using `rusqlite` to query the Bear note database.
* Some thoughts after I finished the first iteration of the application.


## Setting up a CLI

After checking out a couple ways of extracting arguments from the command
line, I settled on [clap](https://clap.rs) mainly due to the ability to
configure the parser using a yaml file. This keeps my rust code focused on
the implementation logic of the utility and moves most of the configuration
boilerplate into a completely separate non-rust file.

For a real life example of how this works, below is the configuration I use
to split the arguments and subcommands with their accompanying
flags/options:

``` yaml
name: cub
version: "0.1.0"
author: Andrew Huynh <a5thuynh@gmail.com>
about: Command-line Utility for Bear.
args:
  - db:
      help: Bear data file to pull data from.
      short: d
      global: true
      takes_value: true
subcommands:
  - ls:
      about: List notes.
      args:
        - all:
            short: a
            help: Show *all* notes.
            conflicts_with: limit
  - show:
      about: Show a single note.
      args:
        - NOTE:
            help: Note ID
            required: true
            index: 1
```

This then enables me to initialize the parser in two very simple lines as seen below:

``` rust
#[macro_use]
extern crate clap;
use clap::App;

fn main() {
    let yaml = load_yaml!("cli.yml");
    let matches = App::from_yaml(yaml).get_matches();
    ...do stuff...
}
```

In addition to setting up the subcommands and flags, `clap` also generates
useful documentation that shows up when you run `--help`.

![CUB example](/img/2018/cub-help.gif)


## Speaking to the Bear

Great, now that we have a way to take subcommands and arguments, lets look
into how we can list out notes and run some basic queries. The Bear app
stores the notes and associated metadata in a sqlite database, I’m assuming
due to the use of Apple’s CoreData frameworks.

To keep things safe, this first version will only do read only actions.
With a little poking around I was able to determine that the local storage
for note data is under:

```
$HOME/Library/Containers/net.shinyfrog.bear/Data/Documents/Application Data
```


### Reverse Engineering the Data Format

Luckily, if you know your way around sqlite, it’s trivial to look through
how the application stores its data. Let's start with determining how notes
and the associated tags are stored. We can list out all the tables in a
database file using the `.tables` command.

``` bash
> sqlite3 database.sqlite
sqlite> .tables
ZSFCHANGE      ZSFLOCATION    ZSFNOTETAG     ZSFURL
Z_MODELCACHE   ZSFCHANGEITEM  ZSFNOTE        ZSFSTATICNOTE
Z_5TAGS        Z_PRIMARYKEY   ZSFFOLDER      ZSFNOTEFILE
ZSFTODO        Z_METADATA
sqlite>
```

Here we can see some potentially useful tables to rifle through. `ZSFNOTE`
looks the most promising and we can use the `.schema` command to determine
the structure of the table.

``` bash
sqlite> .schema ZSFNOTE
CREATE TABLE ZSFNOTE (
    Z_PK INTEGER PRIMARY KEY,
    ...
    ZARCHIVEDDATE TIMESTAMP,
    ZCREATIONDATE TIMESTAMP,
    ...
    ZSUBTITLE VARCHAR,
    ZTEXT VARCHAR,
    ZTITLE VARCHAR,
    ...
);
```
***NOTE:** I left out some columns to make the schema more readable*

Perfect! This has the entire note text data as well as some useful flags
for that particular note that we can use to apply filters to our CLI.
Looking at the schema of `ZSFNOTETAG` and `Z_5TAGS` we also have the
connection between notes and tags.


### Running Queries

To connect to the database, I’m using the
[rusqlite](https://github.com/jgallagher/rusqlite) framework. Setting up
`rusqlite` is very straightforward. Here’s a small example where I connect
to the database and run a simple select query on the notes table just to
really hammer home how simple it is.

``` rust
use rusqlite::{ Connection };
use libcub::note::{ Note };

fn connect_to_db(datafile: &str) -> Connection {
    return Connection::open(datafile).unwrap();
}

fn main() {
    let conn = connect_to_db(<path>);
    let mut stmt = conn.prepare("SELECT * FROM ZSFNOTE").unwrap();
    let notes = stmt.query_map(&[&val], |row| {
        Note::from_sql(row)
    }).unwrap();
    for note in notes {
        println!("{:?}", note);
    }
}
```

And thats all there is to it!

To support things like filtering by different flags. I set up a rust enum
to represent each filter and modify the query accordingly. For example, when
I added the filters for archived and trashed notes, this is how the query
string was modified:

``` rust
for filter in filters {
    match filter {
        NoteStatus::ARCHIVED => {
            filter_sql.push("ZARCHIVED = 1")
        },
        NoteStatus::TRASHED => {
            filter_sql.push("ZTRASHED = 1")
        },
        NoteStatus::NORMAL => {
            filter_sql.push("(ZARCHIVED = 0 AND ZTRASHED = 0)")
        },
    }
}
```

Each filter is converted into its equivalent SQL syntax and then added to
a vector of filters. This vector is then later on joined to the query
string using a string format.

``` rust
format!("{} WHERE {}", query, filter_sql.join(" OR "));
```

## Learnings and Up Next

This was a short weekend(iso) project that was a lot of fun to work on.
Here are some random learnings that I ran into while working on this:

* `clap` was a pleasure to use and the separate yaml configuration made
  adjusting subcommands and flags a breeze.
* Currently `rusqlite` does not support `IN` queries, such as `SELECT *
  FROM ZSFNOTE WHERE Z_PK IN (<list of numbers)` which made me push
  filtering notes by tag into a later version so I could manually implement
  that query.
