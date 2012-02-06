---
layout: post
title: Publishing a Static Website on CloudFront
short: cfpub
---

My [first CloudFront article](/archives/cloudfront-hosting.html) deals with basic S3 and CloudFront setup and with the DefaultRootObject which is the key of hosting a website on Amazon's CDN.

This second post focuses about getting your website *on* CloudFront by providing maintenance scripts to handle publishing and invalidation. Most of the scripts are generic, but the Rakefile targets a [Jekyll](http://jekyllrb.com)-generated static site.

## The Rakefile

First of all, here's my simple Rakefile. Don't try to use it yet, there's a lot of stuff missing.

<script src="http://gist.github.com/592988.js?file=Rakefile"></script>

There's only 3 commands available:

* server: run the Jekyll server on localhost
* build: generate the website in Jekyll's \_site folder
* **publish**: sync the \_site folder on a S3 bucket and invalidate its content

The publish task is the only one I'll talk about, since the two others are really basic. Before continuing, make sure you have replaced "MYBUCKET" by your own S3 bucket.

## Directory synchronization

I've written a script that wraps the [s3cmd](http://s3tools.org/) "sync" command with an invalidation tool to help with updates:

<script src="http://gist.github.com/592941.js"> </script>

To run this script, s3cmd has to be installed. On OSX, you can do so easily with homebrew:

	$ brew install s3cmd

Now, create an s3.config file (God I hate these) with the *s3cmd --configure* command or copy the following configuration. Don't forget to set your own AWS credentials (rows #2,3).

<script src="http://gist.github.com/592988.js?file=s3.config"></script>

You can test your s3cmd configuration by typing this command to list all your buckets:

	$ s3cmd -c s3.config ls

## Cache invalidation

The sync script invalidates the cache of the published objects by calling this script:

<script src="http://gist.github.com/589132.js"> </script>

Note that you have to set your CloudFront distribution ID and your AWS credentials in the previous script too. Sorry for that, but I didn't feel like refactoring around this painful s3cmd config file.

## Summary and publishing

In your Jekyll working directory, you should now have these files:

* Rakefile: rake tasks to easily manage your site
* aws_cf_sync.rb: script that syncs your local files with your S3 bucket
* aws_cf_invalidate.rb: script that invalidates the cache of the updated files
* s3.config: the configuration of your s3 bucket

Before publishing, I suggest you add these exclusions in your *_config.yml* file:

	exclude: [ Rakefile, aws_cf_sync.rb,
	           aws_cf_invalidate.rb, s3.config ]

If you've been a good reader and followed every instruction, all you have to do to publish your website on your S3 bucket and invalidate the CloudFront cache is:

	$ rake publish

You can now fire up your browser and refresh frantically until you see your changes.
