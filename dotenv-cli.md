# dotenv: read and edit .env files from bash

```shell @mdsh
@module dotenv.md
@main __dotenv
@require bashup/dotenv mdsh-source dotenv.md
```

```shell
.env.-h() { .env.--help "$@"; }
.env.--help() {
	echo "Usage:
  $__dotenv_cmd [-f|--file FILE] COMMAND [ARGS...]
  $__dotenv_cmd -h|--help

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
                           return the new or existing value."
}

__dotenv() {
	set -eu
	__dotenv_cmd=${0##*/}
	.env.export() { .env.parse "$@" || return 0; printf 'export %q\n' "${REPLY[@]}"; REPLY=(); }
	.env "$@" || return $?
	${REPLY[@]+printf '%s\n' "${REPLY[@]}"}
}
```
