## Circle CI Null Bytes in `run:` command's stdin

## Expected:

When I run a command without providing it value to STDIN in circleci, I expect there to be no input.

## Actual

There are 2 null bytes in STDIN

## Context

If you read from STDIN in ruby without providing a stdin then an error should be raised:

```
$ ruby -e "STDIN.read_nonblock(10000)"
Traceback (most recent call last):
  1: from -e:1:in `<main>'
<internal:io>:63:in `read_nonblock': Resource temporarily unavailable - read would block (IO::EAGAINWaitReadable)
```

This is because there's nothing in STDIN.

## Reproduction

When you execute a raw read from STDIN on CircleCI you instead get this:

```
  #!/bin/bash -eo pipefail
ruby -e "require \"io/console\"; STDIN.raw!; puts STDIN.read_nonblock(1000).inspect"
"\x00\x00"
```

> Note: There's two null characters here, there should be nothing or an IO::EAGAINWaitReadable error.

To get this output run:

```
$ circleci local execute
```

## Extra info

It looks like these bytes get passed to every `run:` command, if you add multiple `run` commands it happens every time:

```
version: 2.1

orbs:
  heroku: circleci/heroku@1.1.1

jobs:
  build:
    docker:
      - image: circleci/ruby:2.7
    steps:
      - run: 'ruby -e "require \"io/console\"; STDIN.raw!; puts STDIN.read_nonblock(1000).inspect" || exit 0'
      - run: 'ruby -e "require \"io/console\"; STDIN.raw!; puts STDIN.read_nonblock(1000).inspect" || exit 0'
      - run: 'ruby -e "require \"io/console\"; STDIN.raw!; puts STDIN.read_nonblock(1000).inspect" || exit 0'

workflows:
  version: 2.1
  default-ci-workflow:
    jobs:
      - build
```

Generates:

```
The redacted variables listed above will be masked in run step output.====>> ruby -e "require \"io/console\"; STDIN.raw!; puts STDIN.read_nonblock(1000).inspect" || exit 0
  #!/bin/bash -eo pipefail
ruby -e "require \"io/console\"; STDIN.raw!; puts STDIN.read_nonblock(1000).inspect" || exit 0
"\x00\x00"
====>> ruby -e "require \"io/console\"; STDIN.raw!; puts STDIN.read_nonblock(1000).inspect" || exit 0
  #!/bin/bash -eo pipefail
ruby -e "require \"io/console\"; STDIN.raw!; puts STDIN.read_nonblock(1000).inspect" || exit 0
"\x00\x00"
====>> ruby -e "require \"io/console\"; STDIN.raw!; puts STDIN.read_nonblock(1000).inspect" || exit 0
  #!/bin/bash -eo pipefail
ruby -e "require \"io/console\"; STDIN.raw!; puts STDIN.read_nonblock(1000).inspect" || exit 0
"\x00\x00"
Success!
```
