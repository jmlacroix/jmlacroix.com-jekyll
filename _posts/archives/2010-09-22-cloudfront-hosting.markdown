---
layout: post
title: Hosting a Static Website on CloudFront
short: cfhost
nofoot: true
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

<p><a class="src" href="https://gist.github.com/591196">#</a></p>

<pre>
<span class="PreProc">require</span> <span class="Special">'</span><span class="String">rubygems</span><span class="Special">'</span>
<span class="PreProc">require</span> <span class="Special">'</span><span class="String">hmac-sha1</span><span class="Special">'</span>
<span class="PreProc">require</span> <span class="Special">'</span><span class="String">net/https</span><span class="Special">'</span>
<span class="PreProc">require</span> <span class="Special">'</span><span class="String">base64</span><span class="Special">'</span>

s3_access=<span class="Special">'</span><span class="String">S3_ACCESS_KEY</span><span class="Special">'</span>
s3_secret=<span class="Special">'</span><span class="String">S3_SECRET_KEY</span><span class="Special">'</span>
cf_distribution=<span class="Special">'</span><span class="String">CLOUDFRONT_DISTRIBUTION_ID</span><span class="Special">'</span>

newobj = <span class="Identifier">ARGV</span>[<span class="Constant">0</span>]

<span class="Statement">if</span> newobj == <span class="Constant">nil</span>
  puts <span class="Special">&quot;</span><span class="String">usage: aws_cf_setroot.rb index.html</span><span class="Special">&quot;</span>
  <span class="Statement">exit</span>
<span class="Statement">end</span>

date = <span class="Type">Time</span>.now.utc
date = date.strftime(<span class="Special">&quot;</span><span class="String">%a, %d %b %Y %H:%M:%S %Z</span><span class="Special">&quot;</span>)
digest = <span class="Type">HMAC</span>::<span class="Type">SHA1</span>.new(s3_secret)
digest &lt;&lt; date

uri = <span class="Type">URI</span>.parse(<span class="Special">'</span><span class="String"><a href="https://cloudfront.amazonaws.com/2010-08-01/distribution/">https://cloudfront.amazonaws.com/2010-08-01/distribution/</a></span><span class="Special">'</span> + cf_distribution + <span class="Special">'</span><span class="String">/config</span><span class="Special">'</span>)

req = <span class="Type">Net</span>::<span class="Type">HTTP</span>::<span class="Type">Get</span>.new(uri.path)

req.initialize_http_header({
  <span class="Special">'</span><span class="String">x-amz-date</span><span class="Special">'</span> =&gt; date,
  <span class="Special">'</span><span class="String">Content-Type</span><span class="Special">'</span> =&gt; <span class="Special">'</span><span class="String">text/xml</span><span class="Special">'</span>,
  <span class="Special">'</span><span class="String">Authorization</span><span class="Special">'</span> =&gt; <span class="Special">&quot;</span><span class="String">AWS %s:%s</span><span class="Special">&quot;</span> % [s3_access, <span class="Type">Base64</span>.encode64(digest.digest)]
})

http = <span class="Type">Net</span>::<span class="Type">HTTP</span>.new(uri.host, uri.port)
http.use_ssl = <span class="Boolean">true</span>
http.verify_mode = <span class="Type">OpenSSL</span>::<span class="Type">SSL</span>::<span class="Type">VERIFY_NONE</span>
res = http.request(req)

currentobj = <span class="Special">/</span><span class="String">&lt;DefaultRootObject&gt;</span><span class="Special">(</span><span class="Special">.</span><span class="Special">*?</span><span class="Special">)</span><span class="String">&lt;</span><span class="Special">\/</span><span class="String">DefaultRootObject&gt;</span><span class="Special">/</span>.match(res.body)[<span class="Constant">1</span>]

<span class="Statement">if</span> newobj == currentobj
  puts <span class="Special">&quot;</span><span class="String">'</span><span class="Special">#{</span>currentobj<span class="Special">}</span><span class="String">' is already the DefaultRootObject</span><span class="Special">&quot;</span>
  <span class="Statement">exit</span>
<span class="Statement">end</span>

etag = res.header[<span class="Special">'</span><span class="String">etag</span><span class="Special">'</span>]

req = <span class="Type">Net</span>::<span class="Type">HTTP</span>::<span class="Type">Put</span>.new(uri.path)

req.initialize_http_header({
  <span class="Special">'</span><span class="String">x-amz-date</span><span class="Special">'</span> =&gt; date,
  <span class="Special">'</span><span class="String">Content-Type</span><span class="Special">'</span> =&gt; <span class="Special">'</span><span class="String">text/xml</span><span class="Special">'</span>,
  <span class="Special">'</span><span class="String">Authorization</span><span class="Special">'</span> =&gt; <span class="Special">&quot;</span><span class="String">AWS %s:%s</span><span class="Special">&quot;</span> % [s3_access, <span class="Type">Base64</span>.encode64(digest.digest)],
  <span class="Special">'</span><span class="String">If-Match</span><span class="Special">'</span> =&gt; etag
})

<span class="Statement">if</span> currentobj == <span class="Constant">nil</span>
  regex = <span class="Special">/</span><span class="String">&lt;</span><span class="Special">\/</span><span class="String">DistributionConfig&gt;</span><span class="Special">/</span>
  replace = <span class="Special">&quot;</span><span class="String">&lt;DefaultRootObject&gt;</span><span class="Special">#{</span>newobj<span class="Special">}</span><span class="String">&lt;/DefaultRootObject&gt;&lt;/DistributionConfig&gt;</span><span class="Special">&quot;</span>
<span class="Statement">else</span>
  regex = <span class="Special">/</span><span class="String">&lt;DefaultRootObject&gt;</span><span class="Special">(</span><span class="Special">.</span><span class="Special">*?</span><span class="Special">)</span><span class="String">&lt;</span><span class="Special">\/</span><span class="String">DefaultRootObject&gt;</span><span class="Special">/</span>
  replace = <span class="Special">&quot;</span><span class="String">&lt;DefaultRootObject&gt;</span><span class="Special">#{</span>newobj<span class="Special">}</span><span class="String">&lt;/DefaultRootObject&gt;</span><span class="Special">&quot;</span>
<span class="Statement">end</span>

req.body = res.body.gsub(regex, replace)

http = <span class="Type">Net</span>::<span class="Type">HTTP</span>.new(uri.host, uri.port)
http.use_ssl = <span class="Boolean">true</span>
http.verify_mode = <span class="Type">OpenSSL</span>::<span class="Type">SSL</span>::<span class="Type">VERIFY_NONE</span>
res = http.request(req)

puts res.code
puts res.body
</pre>

You need to edit 3 configuration lines at the top of the file:

1. Set s3\_access to your S3 access key
2. Set s3\_secret to your S3 secret key
3. Set cf\_distribution to your CloudFront distribution ID (get it in the properties pane of your "Amazon CloudFront" tab in the AWS Console)

You are now ready to set your DefaultRootObject by running the script with the name of the file you want for root object as parameter:

	$ ruby aws_cf_setroot.rb index.html

index.html is now the default root object (meaning it won't work for subfolders). There may be a small delay before it works, but if you go to yourdomain.com in a web browser, you should be shown your default index.html page.

<footer>
  <a href="/">Jean-Michel Lacroix</a> &ndash;
  <time datetime="{{ page.date | date_to_xmlschema }}" pubdate="pubdate">{{ page.date | date: "%B %d, %Y" }}</time>
</footer>

<section class="footnotes">
<ol>
<li id="ffn1">That could be fixed with a second bucket with the exact same html files that meta refreshes the page to the correct URL. That's overkill, but it should be good enough for basic SEO. <a href="#fn1" title="Jump back to footnote 1 in the text.">&#8617;</a></li>
<li id="ffn2">You could skip this step, but you need to make sure you override the ACLs every time you update your site. <a href="#fn2" title="Jump back to footnote 2 in the text.">&#8617;</a></li>
<li id="ffn3">Not all DNS providers can do that. I'm currently using namecheap.com which allows it. <a href="#fn3" title="Jump back to footnote 3 in the text.">&#8617;</a></li>
</ol>
</section>

[1]: #ffn1
[2]: #ffn2
[3]: #ffn3
