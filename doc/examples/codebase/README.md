# Redemption Project Codebase Demo

This example demonstrates repairing a multiple-file codebase with the Redemption tool

## Copyright

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

This is a demo scenario involving the Redemption of False Positives project.  Unless stated otherwise, any shell commands you execute should be done in this directory (`doc/examples/codebase`)

## Running the Redemption Tool
### Source Code

We are using the `dos2unix` codebase, version 7.5.2. It is freely available from [dos2unix.sourceforge.io/](https://sourceforge.net/projects/dos2unix/files/dos2unix/7.5.2/dos2unix-7.5.2.tar.gz/download).  For this demo, you should download `dos2unix-7.5.2`, unpack it, and place it in a `dos2unix-7.5.2` sub-directory of this directory (`doc/examples/codebase`).

### The `good` directory

This directory contains output for each of the steps below. As you run each step, you should compare your current directory's file contents with the good directory. Assuming your step was run correctly, there will be no differences between the files that appear in both your directory and the good directory.

``` sh
diff -ru . good
```

### Creating a build file

For the rest of these instructions, you should execute the commands inside the `distrib` Docker container, and `cd` into this directory.  To set this up, use this command:

``` sh
docker run -it --rm  -v ${PWD}:/host  docker.cc.cert.org/redemption/distrib  bash
```

Then, inside the shell this command gives you:

``` sh
pushd doc/examples/codebase
```

The following command, when run in the `distrib` container, creates the `compile_commands.json` file for dos2unix:

``` sh
pushd dos2unix-7.5.2
bear -- make CC=clang
make clean
mv compile_commands.json  ..
popd
```

### Static Analysis

Produce static analysis on the given C file, using Clang-tidy 16.0.6, which lives in the Redemption container.  Use this command:

``` sh
pushd dos2unix-7.5.2
clang-tidy -checks='*' *.c  > ../clang-tidy.txt
popd
```

### Convert to Redemption Input

The next step is to convert the `cppcheck.xml` format into a simple JSON format that the redemption tool understands. The `alerts2input.py` file produces suitable JSON files. So you must run this script first; it will create the `alerts.json` file with the alerts you will use.

``` sh
python3 /host/code/analysis/alerts2input.py  ${PWD}/dos2unix-7.5.2  clang_tidy_oss  clang-tidy.txt  alerts.json
```

### Execution

Here is an example of how to run a built-in end-to-end automated code repair test, within the container (you can change the `out` directory location or directory name, but you must create that directory before running the command):

```sh
EXAMPLE=/host/doc/examples/codebase
pushd /host/code/acr
python3 sup.py  -c ${EXAMPLE}/compile_commands.json  -a ${EXAMPLE}/alerts.json  -b ${EXAMPLE}/dos2unix-7.5.2  --repaired-src ${EXAMPLE}/out
popd
```

You can see the repairs using this command:

```sh
diff -ru dos2unix-7.5.2 out
```