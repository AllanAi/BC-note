## 遍历 find 返回的文件名

As long as no file or folder in the subtree has whitespace in its name, you can just loop over the files:

```sh
for i in $(find -name \*.txt); do # Not recommended, will break on whitespace
    process "$i"
done
```

Three reasons of not use this:

- For the for loop to even start, the `find` must run to completion.
- If a file name has any whitespace (including space, tab or newline) in it, it will be treated as two separate names.
- Although now unlikely, you can overrun your command line buffer. Imagine if your command line buffer holds 32KB, and your `for` loop returns 40KB of text. That last 8KB will be dropped right off your `for` loop and you'll never know it.

It is *much* better to glob when you can. White-space safe, for files in the current directory:

```sh
for i in *.txt; do # Whitespace-safe but not recursive.
    process "$i"
done
```

By enabling the `globstar` option, you can glob all matching files in this directory and all subdirectories:

```sh
# Make sure globstar is enabled
shopt -s globstar
for i in **/*.txt; do # Whitespace-safe and recursive
    process "$i"
done
```

`read` can be used safely in combination with `find` by setting the delimiter appropriately:

```sh
# IFS= makes sure it doesn't trim leading and trailing whitespace
# -r prevents interpretation of \ escapes.
# The -print0 will use the NULL as a file separator instead of a newline and the -d '' will use NULL as the separator while reading.
find . -name '*.txt' -print0 | while IFS= read -r -d '' file; do 
    process "$file"
done
```

For more complex searches, you will probably want to use `find`, either with its `-exec` option or with `-print0 | xargs -0`:

```sh
# execute `process` once for each file
find . -name \*.txt -exec process {} \;
# execute `process` once with all the files as arguments*:
find . -name \*.txt -exec process {} +
# using xargs*
find . -name \*.txt -print0 | xargs -0 process
# using xargs with arguments after each filename (implies one run per filename)
find . -name \*.txt -print0 | xargs -0 -I{} process {} argument
```

`find` can also cd into each file's directory before running a command by using `-execdir` instead of `-exec`, and can be made interactive (prompt before running the command for each file) using `-ok` instead of `-exec` (or `-okdir` instead of `-execdir`).

https://stackoverflow.com/questions/9612090/how-to-loop-through-file-names-returned-by-find

## Use Grep to Find Files Based on Content

https://www.linode.com/docs/tools-reference/tools/find-files-in-linux-using-the-command-line/

The `find` command is only able to filter the directory hierarchy based on a file’s name and meta data. If you need to search based on the content of the file, use a tool like [grep](http://localhost:1313/docs/tools-reference/search-and-filter-text-with-grep). Consider the following example:

```
find . -type f -exec grep "example" '{}' \; -print
```

This searches every object in the current directory hierarchy (`.`) that is a file (`-type f`) and then runs the command `grep "example"` for every file that satisfies the conditions. The files that match are printed on the screen (`-print`). The curly braces (`{}`) are a placeholder for the `find` match results. The `{}` are enclosed in single quotes (`'`) to avoid handing `grep` a malformed file name. The `-exec` command is terminated with a semicolon (`;`), which should be escaped (`\;`) to avoid interpretation by the shell.

Before the implementation of the `-exec` option, this kind of command might have used the `xargs` command to generate a similar output:

```
find . -type f -print | xargs grep "example"
```

## Optimization for Find

`find` optimizes its filtering strategy to increase performance. Three user-selectable optimization levels are specified as `-O1`, `-O2`, and `-O3`. The `-O1` optimization is the default and forces `find` to filter based on filename before running all other tests.

Optimization at the `-O2` level prioritizes file name filters, as in `-O1`, and then runs all file-type filtering before proceeding with other more resource-intensive conditions. Level `-O3` optimization allows `find` to perform the most severe optimization and reorders all tests based on their relative expense and the likelihood of their success.

