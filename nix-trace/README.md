# trace-nix

Using `LD_PRELOAD` trick to trace nix access to dirs and files.

## Usage

Run `nix-shell` (or `nix repl` or any other nix tool) with the `TRACE_NIX` environment variable.

Example:
``` bash
LD_PRELOAD=/path/to/trace-nix.so TRACE_NIX=./log nix-shell -p stdenv --run :

# The NUL-separated list of entries will be stored in `./log`
```

## Log format

Since the file names could contain arbitrary byte sequences (broken utf8, `\n`, etc), the NUL-separated format is chosen.

```
lstat() == -1:               `s` FILENAME `\0` `-` `\0`
lstat() ==  0 && S_ISLNK():  `s` FILENAME `\0` `l` readlink(FILENAME) `\0`
lstat() ==  0 && S_ISDIR():  `s` FILENAME `\0` `d` `\0`
lstat() ==  0 && !S_ISDIR() && !S_ISLNK(): `s` FILENAME `\0` `+` `\0`

open() == -1:                `f` FILENAME `\0` `-` `\0`
open() != -1:                `f` FILENAME `\0` b3sum(file contents) `\0`
open() != -1 && read error:  `f` FILENAME `\0` `e` `\0`

opendir() == NULL:           `d` FILENAME `\0` `-` `\0`
opendir() != NULL:           `d` FILENAME `\0` b3sum(directory listing) `\0`

mkdir() == 0                 `t` FILENAME `\0` `+` `\0`
unlinkat() == NULL           `t` FILENAME `\0` `-` `\0`
```

Directory listing:
```
find -mindepth 1 -maxdepth 1 -printf '%P=%y\0' | sed -z 's/[^dlf]$/u/' | LC_ALL=C sort -z
```
