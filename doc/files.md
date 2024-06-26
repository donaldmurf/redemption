# How Files are Organized in This Project

<legal>
'Redemption' Automated Code Repair Tool
Copyright 2023, 2024 Carnegie Mellon University.
NO WARRANTY. THIS CARNEGIE MELLON UNIVERSITY AND SOFTWARE ENGINEERING
INSTITUTE MATERIAL IS FURNISHED ON AN 'AS-IS' BASIS. CARNEGIE MELLON
UNIVERSITY MAKES NO WARRANTIES OF ANY KIND, EITHER EXPRESSED OR IMPLIED,
AS TO ANY MATTER INCLUDING, BUT NOT LIMITED TO, WARRANTY OF FITNESS FOR
PURPOSE OR MERCHANTABILITY, EXCLUSIVITY, OR RESULTS OBTAINED FROM USE OF
THE MATERIAL. CARNEGIE MELLON UNIVERSITY DOES NOT MAKE ANY WARRANTY OF ANY
KIND WITH RESPECT TO FREEDOM FROM PATENT, TRADEMARK, OR COPYRIGHT
INFRINGEMENT.
Licensed under a MIT (SEI)-style license, please see License.txt or
contact permission@sei.cmu.edu for full terms.
[DISTRIBUTION STATEMENT A] This material has been approved for public
release and unlimited distribution.  Please see Copyright notice for
non-US Government use and distribution.
This Software includes and/or makes use of Third-Party Software each
subject to its own license.
DM23-2165
</legal>

All of the files in your public redemption directory can be partitioned as follows:

## Files not known to Git

The `git status` command will list all files that Git does not know about. (If Git knows nothing about all of the files in a directory, it will list that directory as unknown.)

## Files that Git intentionally ignores

The `.gitignore` file contains glob patterns to indicate all files ignored by Git.

## Files known to Git

You can browse these files at the appropriate Bitbucket location:
https://bitbucket.cc.cert.org/bitbucket/projects/REM/repos/redemption.public/browse
or on Github:
https://github.com/cmu-sei/redemption

These files do get released. They also get tested by our CI system.

Most of the files that are human-readable must be adorned with copyright statements. This is accomplished by the `update_copyright.py` script.  Each file that needs a copyright must have the following tags: `<legal>` and `</legal>`. These tags should be 'commented-out' using the file's convention for comments. The `update_markings.py` script injects the current copyright test inside these `<legal>` tags. The copyright text lives in the `ABOUT` file.

If you create a new human-readable file, you should endow it with `<legal>` and `</legal>` tags, near the top of the file. Generally, the tags should go after the title, or after a comment indicating what the file is.  You could then run `update_markings.py` to provide the copyright info. Or you could copy the legal tags from a suitable pre-existing file. There are pytests in `update_markings.py` which will fail if a file should have legal tags, but lacks them.

Consequently all the files in the repository can be partitioned as follows:

#### Non-human-readable files

These files are indicated by their suffix (.e.g. `jpeg`, `mp3`, etc)

#### Files with legal tags

The `grep` command can tell you which files have `<legal>` tags.

#### Files that need not contain legal tags

The `update_markings.yml` file currently lists a set of files that could (in theory) have `<legal>` tags but do not need them.

#### Files that should have legal tags but don't

If a file is human-readable, lacks `<legal>` tags, and is not listed as exempt from `<legal>` tags, this is an error. To see which files should have `<legal>` tags but don't, use this command:

```shell
pytest update_markings.py
```
