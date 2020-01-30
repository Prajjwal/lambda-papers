---
layout: post
title: Null Terminated Lists of Things in Bash
date: 2020-01-29
---

You know the drill. You have to do something with a bunch of files, so you
`find` them and use `xargs` to do what you will.

```zsh
find -iname '*.tmp' | xargs rm
```

`xargs` assumes whitespace separated input by default, which means that if
there's a `this file.tmp` in there, `xargs` tries to do the following:

```zsh
rm 'this'
rm 'file.tmp'
```

That fails if you're lucky. If you aren't, some of those files really do exist
and get unintentionally deleted.

A favorable way of doing this is to remove ambiguity from the list of files
you're passing around by making sure each filename is terminated by a [null
byte](https://en.wikipedia.org/wiki/Null_character). Because filenames on Linux
can never contain `NUL`, programs can safely assume that any characters
encountered before it are part of the filename and *not* a possible delimiter.
The following works correctly for filenames that contain whitespace or newlines:

```zsh
find -iname '*.tmp' -print0 | xargs -0 rm
```

This is a standard idiom in any command line user's repertoire.

`find` and `xargs` aren't the only two tools you're used to that support this,
though, and that lets you do some cool things with confidence.

**Note:** There are arguably better ways of doing this, such as using `find
-exec` or [GNU Parallel](https://www.gnu.org/software/parallel/) in place of
`xargs`.

## Example: Passing around filenames without their extension

You usually want to do this when you're planning on converting one type of file
to another. The `basename` utility that lets us strip a trailing suffix from a
string supports `NUL` terminated output with the `-z` or `--zero` flags. I
commonly convert images and video as follows:

```bash
# Repackage all '.mp4' files in the current directory as '.mkv'
basename -azs .mp4 *.mp4 | xargs -0 -I{} ffmpeg -i {}.mp4 \
                           -acodec copy \
                           -vcodec copy \
                           {}.mkv
```

```bash
# Convert all '.png' files in the current directory to '.jpg' with the same
# name.
basename -azs .jpg *.jpg | xargs -0 -I{} convert {}.png {}.jpg
```

## Example: Passing around files containing a search pattern

Because every single program must come up with its own name for the same thing,
`grep` will output a null terminated list of files if you give it the `-Z` or
`--null` flags on the command line. Use that to delete all files in the current
working directory containing the word 'consistency' as follows:

```bash
grep -s -lZ 'consistency' ./* | xargs -0 rm
```

## Keeping track of it all

Because there's so many different flags for this, I find it helpful to mentally
group programs that use the same one. Eg. the `sort`, `uniq`, `sed`, and
`basename` utilities all use `-z`.

Be warned, these can still have different multi character flags for the same
thing:

* `basename --zero`
* `sort --zero-terminated`
* `uniq --zero-terminated`
* `sed --null-data`

This is why I drink.

~
