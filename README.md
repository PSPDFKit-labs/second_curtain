Second Curtain
=============================

[![Build Status](https://travis-ci.org/AshFurrow/second_curtain.svg?branch=master)](https://travis-ci.org/AshFurrow/second_curtain)

If you're using the cool [FBSnapshotTestCase](https://github.com/facebook/ios-snapshot-test-case) to test your iOS view logic, awesome! Even better if you have continuous integration, like on [Travis](https://travis-ci.org), to automate running those tests!

Purpose
----------------

Isn't it frustrating that you can't *see* the results of your tests? At best, you'll get this kind of error output:

``` sh
ASHViewControllerSpec
  a_view_controller_with_a_loaded_view_should_have_a_valid_snapshot, expected a matching snapshot in a_view_controller_with_a_loaded_view_should_have_a_valid_snapshot
  /Users/travis/build/AshFurrow/upload-ios-snapshot-test-case/Demo/DemoTests/DemoTests.m:31

        it(@"should have a valid snapshot", ^{
            expect(viewController).to.haveValidSnapshot();
        });

    Executed 1 test, with 1 failure (1 unexpected) in 0.952 (0.954) seconds
** TEST FAILED **
```

Wouldn't it be awesome if we could upload the failing test snapshots somewhere, so we can see exactly what's wrong? That's what we aim to do here.

Project Status
----------------

First version complete. There are still a few things up in the air, but it should be settled down soon.

Usage
----------------

Usage is pretty simple. Have an S3 bucket that is world-readable (that is, include the following bucket policy).

``` json
{
  "Version": "2008-10-17",
  "Statement": [
    {
      "Sid": "AllowPublicRead",
      "Effect": "Allow",
      "Principal": {
        "AWS": "*"
      },
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::bucket-name/*"
    }
  ]
}
```

(Replace "bucket-name" with your bucket name.)

It's also a good idea not to use a general-purpose S3 user for your CI, so create a new one with the following policy that will let them list buckets, but only read from or write to the bucket you're using.

``` json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:ListAllMyBuckets"],
      "Resource": "arn:aws:s3:::*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "s3:*"
      ],
      "Resource": "arn:aws:s3:::bucket-name"
    },
    {
      "Effect": "Allow",
      "Action": [
        "s3:*"
      ],
      "Resource": "arn:aws:s3:::bucket-name/*"
    }
  ]
}
```

OK, so the hard part is mostly done. Now that we have a place to upload our images, let's configure our build.

I use a Makefile to build and run tests on my iOS project, so my `.travis.yml` file looks something like this:

``` ruby
language: objective-c
cache: bundler

env:
  - UPLOAD_IOS_SNAPSHOT_BUCKET_NAME=bucket-name
  - UPLOAD_IOS_SNAPSHOT_BUCKET_PREFIX=an/optional/prefix
  - AWS_ACCESS_KEY_ID=ACCESS_KEY
  - AWS_SECRET_ACCESS_KEY=SECRET_KEY

before_install:
  - bundle install

before_script:
  - export LANG=en_US.UTF-8
  - make ci

script:
  - make test
```

(You can take a look at how to [encrypt information in your config file](http://docs.travis-ci.com/user/encryption-keys/), but this has limitations due to how ecrypted variables are accessed via PRs on forks.)

My Makefile looks like this:

``` sh
WORKSPACE = Demo/Demo.xcworkspace
SCHEME = Demo

all: ci

build:
	set -o pipefail && xcodebuild -workspace $(WORKSPACE) -scheme $(SCHEME) -sdk iphonesimulator build | xcpretty -c

clean:
	xcodebuild -workspace $(WORKSPACE) -scheme $(SCHEME) clean

test:
	set -o pipefail && xcodebuild -workspace $(WORKSPACE) -scheme $(SCHEME) -configuration Debug test -sdk iphonesimulator | second_curtain | xcpretty -c --test

ci:	build
```

Notice that we're piping the output from `xcodebuild` into `second_curtain`.

Note also that we're using [`xcpretty`](http://github.com/supermarin/xcpretty), as you should, too. The `xcpretty` invocation must come *after* the `second_curtain` invocation, since Second Curtain relies on parsing the output from `xcodebuild` directly.

And finally, our Gemfile:

```
source 'https://rubygems.org'

gem 'cocoapods'
gem 'xcpretty'
gem 'second_curtain', '~> 0.2'
```

And when any snapshot tests fail, they'll be uploaded to S3 and an HTML page will be generated with links to the images so you can download them. Huzzah!

Note that when the S3 directory is created, it needs to be *unique*. You can provide a custom folder name with the `UPLOAD_IOS_SNAPSHOT_FOLDER_NAME` environment variable. If one is not provided, Second Curtain will fall back to `TRAVIS_JOB_ID`, and then onto one generated by the current date and time.
