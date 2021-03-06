# Setting up New Project

Fuzzer configuration files are placed in a subdirectory inside the [`projects/` dir](../projects) 
of the [OSS-Fuzz repo] on GitHub. 
For example, the configuration files for the
[boringssl](https://github.com/google/boringssl) project are located in 
[`projects/boringssl`](../projects/boringssl).

## Prerequisites
- [Install Docker](https://docs.docker.com/engine/installation). ([Why Docker?](faq.md#why-do-you-use-docker))
- [Integrate](ideal_integration.md) one or more [Fuzz Targets](http://libfuzzer.info/#fuzz-target)
  with the project you want to fuzz.

## Overview

To add a new OSS project to OSS-Fuzz, 3 supporting files have to be added to OSS-Fuzz repository:

* `projects/<project_name>/Dockerfile` - defines a container environment with all the dependencies
needed to build the project and its fuzzers.
* `projects/<project_name>/build.sh` - build script that executes inside the container.
* `projects/<project_name>/project.yaml` - provides metadata about the project.

To create a new directory for the project and 
*automatically generate* template versions of these files, run the following commands:

```bash
$ cd /path/to/oss-fuzz
$ export PROJECT_NAME=project_name
$ python infra/helper.py generate $PROJECT_NAME
```

Create a fuzzer and add it to the *project_name/* directory as well.

## Dockerfile

This is the Docker image definition that build.sh will be executed in.
It is very simple for most projects:
```docker
FROM ossfuzz/base-libfuzzer               # base image with clang toolchain
MAINTAINER YOUR_EMAIL                     # each file should have a maintainer
RUN apt-get install -y ...                # install required packages to build a project
RUN git checkout <git_url> <checkout_dir> # checkout all sources needed to build your project
WORKDIR <checkout_dir>                    # current directory for build script
COPY build.sh fuzzer.cc $SRC/             # install build script and other source files.
```
Expat example: [expat/Dockerfile](../projects/expat/Dockerfile)

### Fuzzer execution environment

[This page](fuzzer_environment.md) gives information about the environment that
your fuzzers will run under on ClusterFuzz, and the assumptions that you can
make.

## build.sh

This is where most of the work is done to build fuzzers for your project. The script will
be executed within an image built from a `Dockerfile`.

In general, this script will need to:

1. Build the project using its build system *with* correct compiler and its flags provided as 
  *environment variables* (see below). 
2. Build the fuzzers, linking with the project and libFuzzer. Resulting fuzzers
   should be placed in `/out`.

For expat, this looks like:

```bash
#!/bin/bash -eu

./buildconf.sh
# configure scripts usually use correct environment variables.
./configure

make -j$(nproc) clean all

# build the fuzzer, linking with libFuzzer and libexpat.a
$CXX $CXXFLAGS -std=c++11 -Ilib/ \
    $SRC/parse_fuzzer.cc -o /out/expat_parse_fuzzer \
    -lfuzzer .libs/libexpat.a
```

### build.sh Script Environment 

When build.sh script is executed, the following locations are available within the image:

| Path                   | Description
| ------                 | -----
| `$SRC/<some_dir>`      | Source code needed to build your project.
| `/usr/lib/libfuzzer.a` | Prebuilt libFuzzer library that need to be linked into all fuzzers (`-lfuzzer`).

You *must* use special compiler flags to build your project and fuzzers.
These flags are provided in following environment variables:

| Env Variable           | Description
| -------------          | --------
| `$CC`, `$CXX`, `$CCC`  | The C and C++ compiler binaries.
| `$CFLAGS`, `$CXXFLAGS` | C and C++ compiler flags.

Many well-crafted build scripts will automatically use these variables. If not,
passing them manually to a build tool might be required.

See [Provided Environment Variables](../infra/base-images/base-libfuzzer/README.md#provided-environment-variables) section in 
`base-libfuzzer` image documentation for more details.


## Testing locally

Helper script can be used to build images and fuzzers.

```bash
$ cd /path/to/oss-fuzz
$ python infra/helper.py build_image $PROJECT_NAME
$ python infra/helper.py build_fuzzers $PROJECT_NAME
```

This should place the built fuzzers into `/path/to/oss-fuzz/build/out/$PROJECT_NAME`
directory on your machine (and `/out` in the container). You should then try to run these fuzzers
inside the container to make sure that they work properly:

```bash
$ python infra/helper.py run_fuzzer $PROJECT_NAME name_of_a_fuzzer
```

If everything works locally, then it should also work on our automated builders
and ClusterFuzz.

It's recommended to look at code coverage as a sanity check to make sure that fuzzer gets to the code you expect.

```bash
$ python infra/helper.py coverage $PROJECT_NAME name_of_a_fuzzer
```


## Debugging Problems

[Debugging](debugging.md) document lists ways to debug your build scripts or fuzzers
in case you run into problems.


### Custom libFuzzer options for ClusterFuzz

By default, ClusterFuzz will run your fuzzer without any options. You can specify
custom options by creating a `my_fuzzer.options` file next to a `my_fuzzer` executable in `/out`:

```
[libfuzzer]
max_len = 1024
```

[List of available options](http://llvm.org/docs/LibFuzzer.html#options)

At least, `max_len` is highly recommended which specifies what the maximum length of allowed input to your function.

For out of tree fuzzers, you will likely add options file using docker's
`COPY` directive and will copy it into output in build script. 
([Woff2 example](https://github.com/google/oss-fuzz/blob/master/projects/woff2/convert_woff2ttf_fuzzer.options).)


### Seed Corpus

OSS-Fuzz uses evolutionary fuzzing algorithms. Supplying seed corpus consisting
of sample inputs is one of the best ways to improve fuzzer coverage.

To provide a corpus for `my_fuzzer`, put `my_fuzzer_seed_corpus.zip` file next
to the fuzzer binary in `/out` during the build. Individual files in the zip file 
will be used as starting inputs for mutations. You can store the corpus next to 
source files, generate during build or fetch it using curl or any other tool of 
your choice. 
([Boringssl example](https://github.com/google/oss-fuzz/blob/master/projects/boringssl/build.sh#L42).)

Seed corpus files will be used for cross-mutations and portions of them might appear
in bug reports or be used for further security research. It is important that corpus
has an appropriate and consistent license.


### Dictionaries

Dictionaries hugely improve fuzzer effectiveness for inputs with lots of similar
sequences of bytes. [libFuzzer documentation](http://llvm.org/docs/LibFuzzer.html#dictionaries)

Put your dict file in `/out` and specify in .options file:

```
[libfuzzer]
dict = dictionary_name.dict
```

It is common for several fuzzers to reuse the same dictionary if they are fuzzing very similar inputs.
([Expat example](https://github.com/google/oss-fuzz/blob/master/projects/expat/parse_fuzzer.options).)

## Jenkinsfile

This file will be largely the same for most projects, and is used by our build
infrastructure. For expat, this is:

```groovy
// load libFuzzer pipeline definition.
def libfuzzerBuild = fileLoader.fromGit('infra/libfuzzer-pipeline.groovy',
                                        'https://github.com/google/oss-fuzz.git')

libfuzzerBuild {
  git = "git://git.code.sf.net/p/expat/code_git"
}
```

Simply replace the "git" entry with the correct git url for the project.

*Note*: only git is supported right now.

## Checking in to oss-fuzz repository

Fork OSS-Fuzz, commit and push to the fork, and then create a pull request with
your change! Follow the [Forking Project](https://guides.github.com/activities/forking/) guide
if you are new to contributing via GitHub.

### Copyright headers

Please include copyright headers for all files checked in to oss-fuzz:

```
# Copyright 2016 Google Inc.
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#      http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.
#
################################################################################
```

If porting a fuzzer from Chromium, keep the Chromium license header.

## The end

Once your change is merged, the fuzzers should be automatically built and run on
ClusterFuzz after a short while!

[OSS-Fuzz repo]: https://github.com/google/oss-fuzz
[Dictionaries]: http://llvm.org/docs/LibFuzzer.html#dictionaries
[Install Docker]: https://docs.docker.com/engine/installation/linux/ubuntulinux/
