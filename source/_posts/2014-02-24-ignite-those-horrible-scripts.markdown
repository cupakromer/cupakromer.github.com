---
layout: post
title: "Ignite Those Horrible Scripts"
date: 2014-02-24 11:03
comments: false
categories:
- zsh
- script
- shell
---

The past week was spent dealing with an infuriating data crunching task.
[Chris](https://twitter.com/crsexton) and I decided to celebrate by killing our
poorly written Ruby cli scripts with fire. Printing the scripts and going out
back to set them on fire would be more dramatic. Though it probably would be
appreciated by the other businesses which we share space with.

I took a breather and looked into [zsh](http://www.zsh.org/) functions to
accomplish this. Here's what I came up with:

```sh
rageflip() { echo " ┻━┻ ︵ヽ(\`Д´)ﾉ︵ ┻━┻  $*" }

has_utility() { hash $1 2>/dev/null }

# Wrapper around `rm -rf` expressing callers loathing and disdain for the
# provided files.
#
# Inspiration was Homebrew's output:
# 🍺  /usr/local/Cellar/tmux/1.9: 15 files, 628K, built in 25 seconds
#
# For additional file stats, specific to code files, consider installing the
# Count Lines of Code (cloc) tool: http://cloc.sourceforge.net/
#
#   brew install cloc
#   sudo apt-get install cloc
#
ignite() {
  if (( $# == 0 )) then echo "USAGE: ignite file [file] ..."; return; fi
  local total_lines
  local human_size
  local lc_blank
  local lc_comment
  local lc_code
  echo "BURN IT ALL!!! $(rageflip)"
  for i do
    # If the file is empty we have 0 lines
    human_size=$(ls -lh $i | awk '{ print $5 }')
    total_lines=${$(sed -n '$=' $i):-0}
    stats="$total_lines lines"

    if has_utility cloc; then
      # Setup some local variables regarding file stats
      lc_blank=
      lc_comment=
      lc_code=
      eval $(
        cloc $i --quiet | tail -2 | head -1 |
        awk '{ print "lc_blank="$3, "lc_comment="$4, "lc_code="$5 }'
      )
      if [ ! -z $lc_blank ]; then
        stats="$lc_code loc, $lc_comment comments, $lc_blank whitespace lines"
      fi
    fi

    rm -rf $i
    echo "🔥  $i: $stats, $human_size"
  done
}
```

Now we can sit back and watch it all burn:

```sh
$ ignite script/crunch-*
BURN IT ALL!!!  ┻━┻ ︵ヽ(`Д´)ﾉ︵ ┻━┻
🔥  script/crunch-dump-sessions: 5 loc, 5 comments, 2 whitespace lines, 330B
🔥  script/crunch-normalize: 47 loc, 14 comments, 11 whitespace lines, 2.2K
🔥  script/crunch-import-csv: 101 loc, 2 comments, 20 whitespace lines, 2.7K
🔥  script/crunch-timestamp: 203 loc, 44 comments, 38 whitespace lines, 8.4K
```

Though at this point, it might as well be a full script file. If I do that, I
might as well just write it as a Ruby cli...
