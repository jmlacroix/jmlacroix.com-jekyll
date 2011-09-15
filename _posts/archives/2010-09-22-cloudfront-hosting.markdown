---
layout: post
title: Hosting a static website on CloudFront
desc: An overview of the basic Amazon Web Services configuration you need to be able to publish a static site with S3 & CloudFront.
---

I recently tried to use Amazon's CloudFront to host my static [Jekyll](http://jekyllrb.com)-generated homepage. Here's a couple of reasons I really wanted to do that:

* It's dirt cheap
* It's blazing fast
* It's a nice "hack"

By "hack" I mean that it's not really it's purpose, so there are a couple of downsides:

* No error pages (404, etc)
* More complex to setup than most hosting
* No page redirection between www.yourdomain.com and yourdomain.com <sup id="fn1">[1]</sup>

This posts only covers the basic S3 & CloudFront setup you need to host a static website there. In a next post I'll show you how I sync my Jekyll generated files.

## Create a S3 bucket

1. In the AWS Console, go to the "Amazon S3" tab.
2. Use the "Create Bucket" button to create a bucket named MYBUCKET.
3. Right click on your newly created bucket and bring the properties panel.
4. (*Optional <sup id="fn2">[2]</sup>*) Make your bucket public-readable by clicking the "Edit bucket policy" in the "Permissions" tab and adding the following code (don't forget to change MYBUCKET to your bucket name):
		{
		  "Version":"2008-10-17",
		  "Statement":[{
		    "Sid":"AddPerm",
		    "Effect":"Allow",
		      "Principal": {
		  "AWS": "*"
		      },
		      "Action":["s3:GetObject"],
		      "Resource":["arn:aws:s3:::BUCKETNAME/*"]
		    }
		  ]
		}

## Configure a CloudFront distribution

1. In the AWS Console, go to the "Amazon CloudFront" tab.
2. Click on the "Create Distribution" button.
3. Select the MYBUCKET bucket we created earlier as the origin.
4. Specify the CNAMEs your site will be hosting (your site domain name).
5. Back in the CloudFront distributions list, select your newly created distribution and copy it's "Domain Name".

## Edit domain's DNS records

1. Go to your domain's DNS record manager.
2. Set your domain or subdomain so it points to your CloudFront distribution domain name as a CNAME record <sup id="fn3">[3]</sup>.
3. When your DNS finally refreshes (remember, it can be long), you should be able to access your bucket by using your domain or subdomain.

## Setting the DefaultRootObject

The last step is the fun part, since it requires you to run some crafty ruby code.

Since there's no support (yet) in the AWS Console to set the DefaultRootObject (and it's not present in a lot of S3/CloudFront software neither), I've written a small ruby script to allow you to enable it on your distribution.

Here's the code:

<script src="http://gist.github.com/591196.js"> </script>

You need to edit 3 configuration lines at the top of the file:

1. Set s3\_access to your S3 access key
2. Set s3\_secret to your S3 secret key
3. Set cf\_distribution to your CloudFront distribution ID (get it in the properties pane of your "Amazon CloudFront" tab in the AWS Console)

You are now ready to set your DefaultRootObject by running the script with the name of the file you want for root object as parameter:

	$ ruby aws_cf_setroot.rb index.html

index.html is now the default root object (meaning it won't work for subfolders). There may be a small delay before it works, but if you go to yourdomain.com in a web browser, you should be shown your default index.html page.


<div class="footnotes">
<p>Footnotes</p>
<ol>
<li id="ffn1">That could be fixed with a second bucket with the exact same html files that meta refreshes the page to the correct URL. That's overkill, but it should be good enough for basic SEO. <a href="#fn1" title="Jump back to footnote 1 in the text.">&#8617;</a></li>
<li id="ffn2">You could skip this step, but you need to make sure you override the ACLs every time you update your site. <a href="#fn2" title="Jump back to footnote 2 in the text.">&#8617;</a></li>
<li id="ffn3">Not all DNS providers can do that. I'm currently using namecheap.com which allows it. <a href="#fn3" title="Jump back to footnote 3 in the text.">&#8617;</a></li>
</ol>
</div>

[1]: #ffn1
[2]: #ffn2
[3]: #ffn3
