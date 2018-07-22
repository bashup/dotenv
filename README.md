# dotenv: read and edit .env files from bash

.env files are a handy convention for configuring applications (especially with docker-compose).  But when you're writing shell scripts, it can be hard to modify or parse them.  (Especially the ones in [docker-compose format](https://docs.docker.com/compose/compose-file/#env_file)!)

`dotenv` is a tiny bash scripting tool for manipulating `.env` files formatted like this:

```ini
# Lines are comments OR key=value pairs; leading and
# trailing whitespace is stripped
  FOO=bar

# Everything after the '=' is taken EXACTLY AS-IS (except for
# trailing whitespace removal), so the following line sets BAR
# to "baz \"blah # foo".
BAR=baz "blah # foo
```

This format is exactly how docker-compose parses `.env` and `env_files`, so this lets you easily read and write them, without losing comments, changing indentation, or moving/rearranging existing variables.

(Note: if you need to read or write `.env` files that *aren't* in docker-compose syntax (e.g. shell dotfiles), you'll have to escape the values you write and unescape the ones you read, using whatever syntax is compatible with the other tools using the file.  Also, multi-line values can't be used unless the other tools can encode encode/decode linefeeds in such a way that the encoded form doesn't contain any actual linefeeds.)

**Contents**

<!-- toc -->

- [Installation, Requirements And Use](#installation-requirements-and-use)
  * [The `dotenv` CLI](#the-dotenv-cli)
  * [The `.env` shell function](#the-env-shell-function)
  * [File Selection](#file-selection)
    + [`-f` | `--file` *filename*](#-f----file-filename)
  * [Read Operations](#read-operations)
    + [`get` *key*](#get-key)
    + [`parse` *[keys...]*](#parse-keys)
    + [`export` *[keys...]*](#export-keys)
  * [Write Operations](#write-operations)
    + [`set` [+]*key*[=*value*]...](#set-keyvalue)
    + [`puts` *string*](#puts-string)
    + [`generate` *key command [args...]*](#generate-key-command-args)
- [License](#license)

<!-- tocstop -->

## Installation, Requirements And Use

To install dotenv, you can copy and paste the [code](dotenv) into your script, or download it, `chmod +x` it and save it somewhere on your `PATH`.  (if you have [basher](https://github.com/basherpm/basher), you can `basher install bashup/dotenv` to automatically install it.)

The code is licensed CC0, so you are not required to add any attribution or copyright notices to your project.  The only external commands it uses are `cp -p` and `mv`, and it's compatible with bash 3.2+, so it should work even on plain OS X, busybox, or pretty much anything else that can run bash.

Once you've got it installed, there are two ways you can use it: as a command line tool (`dotenv`), or a bash function (`.env`).

### The `dotenv` CLI

To interactively manipulate .env files, you can run `dotenv` as a command line tool, e.g.:

```console
    $ dotenv --help
    Usage:
      dotenv [-f|--file FILE] COMMAND [ARGS...]
      dotenv -h|--help
    
    Options:
      -f, --file FILE          Use a file other than .env
    
    Read Commands:
      get KEY                  Get raw value of KEY (or fail)
      parse [KEY...]           Get trimmed KEY=VALUE lines for named keys (or all)
      export [KEY...]          Export the named keys (or all) in shell format
    
    Write Commands:
      set [+]KEY[=VALUE]...    Set or unset values (in-place w/.bak); + sets default
      puts STRING              Append STRING to the end of the file
      generate KEY [CMD...]    Set KEY to the output of CMD unless it already exists;
                               return the new or existing value.

    $ echo '  # This is my .env file' >prod.env
    $ echo '  FOO=bar  ' >>prod.env

    $ cat prod.env
      # This is my .env file
      FOO=bar  

    $ dotenv -f prod.env get FOO
    bar
```

### The `.env` shell function

To manipulate .env files from within a shell script, you can source `dotenv` and then use the `.env` shell function.  The key differences are that:

* `.env` keeps the last `.env` or `--file` contents in memory for faster reads  (using `--file` again will force a reload)
* `.env` returns results in the bash `$REPLY` variable rather than writing them to standard output, and
* the `export` subcommand actually sets and exports shell variables, rather than outputting `export` commands

```sh
# Source to get the .env function:

    $ source dotenv

# It takes the same arguments as `dotenv`:

    $ .env --file prod.env get FOO

# But output is to the REPLY variable instead of stdout:

    $ echo "$REPLY"
    bar

# And the selected --file is "sticky" from one call to the next:

    $ .env set FOO=wat BAR=baz  # still using --file prod.env

    $ cat prod.env
      # This is my .env file
      FOO=wat
    BAR=baz
```

### File Selection

#### `-f` | `--file` *filename*

By default, all operations read or write to `.env` in the current directory at the time of the operation.  Using this option changes the active file to *filename*, and loads its current contents.  The in-memory contents are then used for any subsequent operations other than `set` (which reloads the file before making any changes).

A non-existent file is silently considered to be empty; file read errors result in a failure return.

### Read Operations

The following operations return their result in `REPLY` (or `REPLY[@]` for multiple results) when called via the `.env` shell function, or output one or more lines to stdout when invoked via the `dotenv` command.  Success is returned if there is at least one result, failure otherwise.

#### `get` *key*

Retrieve the value of *key* from the in-memory file.  Returns success if the key is found, failure otherwise.

```sh
    $ .env get FOO && echo "$REPLY"
    wat

    $ .env get BAZ || echo "no such key"
    no such key
```

#### `parse` *[keys...]*

Return the parsed `KEY=value` pairs from the in-memory file, trimming whitespace and skipping blank lines and comments.  If *keys* are given, only return lines with matching keys (in their in-file order).  If no suitable lines are found, return failure.

```sh
# .env returns results as an array in REPLY:

    $ .env parse
    $ printf '%s\n' "${REPLY[@]}"
    FOO=wat
    BAR=baz

# Restricted to given keys, if any:

    $ .env parse FLIM FOO FLAM; printf '%s\n' "${REPLY[@]}"
    FOO=wat

# Returning failure if there are no matches:

    $ .env parse FLIM FLAM || echo "nothing found"
    nothing found
```

#### `export` *[keys...]*

Export the values found in the file as environment variables (restricted to those named in *keys*, if given).  When used on the command line, output the key-value pairs in a format suitable for `eval`-ing.  Success is returned unless `export` fails due to invalid characters found in a key.

```sh
# dotenv export output is shell-escaped for correct eval-ing:

    $ .env set FOO='bar baz' BING='bang; bong'

    $ dotenv -f prod.env export
    export FOO=bar\ baz
    export BAR=baz
    export BING=bang\;\ bong

# .env export just exports the variables (which can be filtered):

    $ .env export BAR BING   # use -f to force reload from disk

    $ declare -p BAR BING
    declare -x BAR="baz"
    declare -x BING="bang; bong"

    $ declare -p FOO 2>/dev/null || echo "FOO not found"
    FOO not found

    $ .env export  # export everything now

    $ declare -p FOO
    declare -x FOO="bar baz"
```

### Write Operations

#### `set` [+]*key*[=*value*]...

Edit the .env file atomically in place, writing to a `.bak` file and then replacing the original.  (Like sed, this replaces a symlink with a regular file.  (Unless you're using it as a shell function and have also sourced [realpaths](https://github.com/bashup/realpaths), in which case the backup and overwrite are done in the symlinked directory.)

Each argument passed must be match one of the following patterns:

* `KEY=VALUE` sets `KEY` to `VALUE`; if the key previously existed, the line is edited in place, otherwise a new, unindented assignment is appended to the file.

* `+KEY=VALUE` appends `KEY=VALUE` to the file unless there is an existing value for `KEY`:

  ```sh
    $ .env set +FLIM="flam" +FOO="already set, won't change"

    $ dotenv -f prod.env export FLIM FOO
    export FOO=bar\ baz
    export FLIM=flam
  ```

* An argument without an `=` is assumed to be a key that should be deleted from the file, removing any lines that set that key.

  ```sh
  # Unset keys if they exist:

    $ .env set BING FOO FLIM

    $ cat prod.env
      # This is my .env file
    BAR=baz
  ```


Comment lines in the file are unchanged.  The file is reloaded from disk immediately before making the changes, and is only rewritten if the contents actually changed.  (That is, if you set values that already exist or delete values that don't, the file is not rewritten.)

#### `puts` *string*

Append *string* to the .env file *and* its in-memory representation:

```sh
# Append some text:

    $ .env puts '# A new comment'
    $ .env puts 'FLIM=flam'

# The file is updated in memory:

    $ .env get FLIM && echo $REPLY
    flam

# And also on disk:

    $ cat prod.env
      # This is my .env file
    BAR=baz
    # A new comment
    FLIM=flam
```

#### `generate` *key command [args...]*

Sets *key* to the output of running *command args...* unless there is an existing value for *key* (in which case *command* is not run).  The new (or existing) value of *key* is returned, and the result is a success unless *command* was run and failed.  With a suitable *command*, this can be used to generate default passwords, hash keys, etc.

```sh
    $ .env generate FLIM echo spam   # already exists, no change
    $ echo $REPLY                    # returns old value
    flam

    $ .env generate FILE ls   # doesn't exist, so set it to "$(ls)"
    $ echo $REPLY             # returns new value
    prod.env

    $ dotenv -f prod.env get FILE   # and it's updated on disk
    prod.env
```

## License

<p xmlns:dct="http://purl.org/dc/terms/" xmlns:vcard="http://www.w3.org/2001/vcard-rdf/3.0#">
  <a rel="license" href="http://creativecommons.org/publicdomain/zero/1.0/"><img src="https://licensebuttons.net/p/zero/1.0/80x15.png" style="border-style: none;" alt="CC0" /></a><br />
  To the extent possible under law, <a rel="dct:publisher" href="https://github.com/pjeby"><span property="dct:title">PJ Eby</span></a>
  has waived all copyright and related or neighboring rights to <span property="dct:title">bashup/realpaths</span>.
This work is published from: <span property="vcard:Country" datatype="dct:ISO3166" content="US" about="https://github.com/bashup/dotenv">United States</span>.
</p>
