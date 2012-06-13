---
layout: post
title: Publishing a Static Website on CloudFront
short: cfpub
---

My [first CloudFront article](/archives/cloudfront-hosting.html) deals with basic S3 and CloudFront setup and with the DefaultRootObject which is the key of hosting a website on Amazon's CDN.

This second post focuses about getting your website *on* CloudFront by providing maintenance scripts to handle publishing and invalidation. Most of the scripts are generic, but the Rakefile targets a [Jekyll](http://jekyllrb.com)-generated static site.

## The Rakefile

First of all, here's my simple Rakefile. Don't try to use it yet, there's a lot of stuff missing.

<pre>
task <span class="Constant">:default</span> =&gt; <span class="Constant">:server</span>

desc <span class="Special">'</span><span class="String">Start server with --auto</span><span class="Special">'</span>
task <span class="Constant">:server</span> <span class="Statement">do</span>
  jekyll(<span class="Special">'</span><span class="String">--server --auto</span><span class="Special">'</span>)
<span class="Statement">end</span>

desc <span class="Special">'</span><span class="String">Build site with Jekyll</span><span class="Special">'</span>
task <span class="Constant">:build</span> <span class="Statement">do</span>
  jekyll(<span class="Special">'</span><span class="String">--no-future</span><span class="Special">'</span>)
<span class="Statement">end</span>

desc <span class="Special">'</span><span class="String">Build and deploy</span><span class="Special">'</span>
task <span class="Constant">:publish</span> =&gt; <span class="Constant">:build</span> <span class="Statement">do</span>
  bucket = <span class="Special">'</span><span class="String">MYBUCKET</span><span class="Special">'</span>
  puts <span class="Special">&quot;</span><span class="String">Publishing site to bucket </span><span class="Special">#{</span>bucket<span class="Special">}</span><span class="Special">&quot;</span>
  sh <span class="Special">'</span><span class="String">ruby aws_cf_sync.rb _site/ </span><span class="Special">'</span> + bucket
<span class="Statement">end</span>

<span class="PreProc">def</span> <span class="Identifier">jekyll</span>(opts = <span class="Special">''</span>)
  sh <span class="Special">'</span><span class="String">rm -rf _site/*</span><span class="Special">'</span>
  sh <span class="Special">'</span><span class="String">jekyll </span><span class="Special">'</span> + opts
<span class="PreProc">end</span>
</pre>


There's only 3 commands available:

* server: run the Jekyll server on localhost
* build: generate the website in Jekyll's \_site folder
* **publish**: sync the \_site folder on a S3 bucket and invalidate its content

The publish task is the only one I'll talk about, since the two others are really basic. Before continuing, make sure you have replaced "MYBUCKET" by your own S3 bucket.

## Directory synchronization

I've written a script that wraps the [s3cmd](http://s3tools.org/) "sync" command with an invalidation tool to help with updates:

<pre>
local   = <span class="Identifier">ARGV</span>[<span class="Constant">0</span>]
s3_dest = <span class="Identifier">ARGV</span>[<span class="Constant">1</span>]

<span class="Statement">if</span> local == <span class="Constant">nil</span> || s3_dest == <span class="Constant">nil</span>
  puts <span class="Special">&quot;</span><span class="String">syntax aws_cf_sync.rb local_source s3_dest</span><span class="Special">&quot;</span>
  <span class="Statement">exit</span>
<span class="Statement">end</span>

config = <span class="Special">&quot;</span><span class="Special">#{</span><span class="Type">Dir</span>.pwd<span class="Special">}</span><span class="String">/s3.config</span><span class="Special">&quot;</span>
<span class="Statement">if</span> !<span class="Type">File</span>.exists?(config)
  puts <span class="Special">&quot;</span><span class="String">please setup your s3.config file</span><span class="Special">&quot;</span>
  <span class="Statement">exit</span>
<span class="Statement">end</span>

invalidate = <span class="Special">&quot;</span><span class="Special">#{</span><span class="Type">Dir</span>.pwd<span class="Special">}</span><span class="String">/aws_cf_invalidate.rb</span><span class="Special">&quot;</span>
<span class="Statement">if</span> !<span class="Type">File</span>.exists?(invalidate)
  puts <span class="Special">&quot;</span><span class="String">please download the aws_cf_invalidate.rb script</span><span class="Special">&quot;</span>
  <span class="Statement">exit</span>
<span class="Statement">end</span>

s3_dest   = s3_dest.split(<span class="Special">'</span><span class="String">/</span><span class="Special">'</span>)
s3_bucket = s3_dest.shift
s3_path   = s3_dest.join(<span class="Special">'</span><span class="String">/</span><span class="Special">'</span>)

s3_path += <span class="Special">'</span><span class="String">/</span><span class="Special">'</span> <span class="Statement">unless</span> s3_dest.length == <span class="Constant">0</span>

<span class="Special">%x[</span><span class="String"> $(which s3cmd) -c </span><span class="Special">#{</span>config<span class="Special">}</span><span class="String"> sync </span><span class="Special">#{</span>local<span class="Special">}</span><span class="String"> s3://</span><span class="Special">#{</span>s3_bucket<span class="Special">}</span><span class="String">/</span><span class="Special">#{</span>s3_path<span class="Special">}</span><span class="String"> --acl-public </span><span class="Special">]</span>

files = <span class="Special">%x[</span><span class="String"> cd _site &amp;&amp; find . -type f </span><span class="Special">]</span>.split(<span class="Special">&quot;</span><span class="Special">\n</span><span class="Special">&quot;</span>).map <span class="Statement">do</span> |<span class="Identifier">f</span>|
  s3_path + f[<span class="Constant">2</span>,f.length]
<span class="Statement">end</span>

<span class="Special">%x[</span><span class="String"> ruby </span><span class="Special">#{</span>invalidate<span class="Special">}</span><span class="String"> </span><span class="Special">#{</span>files.join(<span class="Special">'</span><span class="String"> </span><span class="Special">'</span>)<span class="Special">}</span><span class="String"> </span><span class="Special">]</span>
</pre>


To run this script, s3cmd has to be installed. On OSX, you can do so easily with homebrew:

	$ brew install s3cmd

Now, create an s3.config file (God I hate these) with the *s3cmd --configure* command or copy the following configuration. Don't forget to set your own AWS credentials (rows #2,3).

<pre>
[default]
access_key = S3_ACCESS_KEY
secret_key = S3_SECRET_KEY
acl_public = False
bucket_location = US
cloudfront_host = cloudfront.amazonaws.com
cloudfront_resource = /2008-06-30/distribution
default_mime_type = binary/octet-stream
delete_removed = False
dry_run = False
encoding = UTF-8
encrypt = False
force = False
get_continue = False
gpg_command = None
gpg_decrypt = %(gpg_command)s -d --verbose --no-use-agent --batch --yes --passphrase-fd %(passphrase_fd)s -o %(output_file)s %(input_file)sgpg_encrypt = %(gpg_command)s -c --verbose --no-use-agent --batch --yes --passphrase-fd %(passphrase_fd)s -o %(output_file)s %(input_file)sgpg_passphrase = 
guess_mime_type = True
host_base = s3.amazonaws.com
host_bucket = %(bucket)s.s3.amazonaws.com
human_readable_sizes = False
list_md5 = False
preserve_attrs = True
progress_meter = True
proxy_host = 
proxy_port = 0 
recursive = False
recv_chunk = 4096
send_chunk = 4096
simpledb_host = sdb.amazonaws.com
skip_existing = False
urlencoding_mode = normal
use_https = False
verbosity = WARNING
</pre>

You can test your s3cmd configuration by typing this command to list all your buckets:

	$ s3cmd -c s3.config ls

## Cache invalidation

The sync script invalidates the cache of the published objects by calling this script:

<pre>
<span class="PreProc">require</span> <span class="Special">'</span><span class="String">rubygems</span><span class="Special">'</span>
<span class="PreProc">require</span> <span class="Special">'</span><span class="String">hmac-sha1</span><span class="Special">'</span>
<span class="PreProc">require</span> <span class="Special">'</span><span class="String">net/https</span><span class="Special">'</span>
<span class="PreProc">require</span> <span class="Special">'</span><span class="String">base64</span><span class="Special">'</span>

s3_access=<span class="Special">'</span><span class="String">S3_ACCESS_KEY</span><span class="Special">'</span>
s3_secret=<span class="Special">'</span><span class="String">S3_SECRET_KEY</span><span class="Special">'</span>
cf_distribution=<span class="Special">'</span><span class="String">CLOUDFRONT_DISTRIBUTION_ID</span><span class="Special">'</span>

<span class="Statement">if</span> <span class="Identifier">ARGV</span>.length &lt; <span class="Constant">1</span>
  puts <span class="Special">&quot;</span><span class="String">usage: aws_cf_invalidate.rb file1.html dir1/file2.jpg ...</span><span class="Special">&quot;</span>
  <span class="Statement">exit</span>
<span class="Statement">end</span>

paths = <span class="Special">'</span><span class="String">&lt;Path&gt;/</span><span class="Special">'</span> + <span class="Identifier">ARGV</span>.join(<span class="Special">'</span><span class="String">&lt;/Path&gt;&lt;Path&gt;/</span><span class="Special">'</span>) + <span class="Special">'</span><span class="String">&lt;/Path&gt;</span><span class="Special">'</span>

date = <span class="Type">Time</span>.now.utc
date = date.strftime(<span class="Special">&quot;</span><span class="String">%a, %d %b %Y %H:%M:%S %Z</span><span class="Special">&quot;</span>)
digest = <span class="Type">HMAC</span>::<span class="Type">SHA1</span>.new(s3_secret)
digest &lt;&lt; date

uri = <span class="Type">URI</span>.parse(<span class="Special">'</span><span class="String"><a href="https://cloudfront.amazonaws.com/2010-08-01/distribution/">https://cloudfront.amazonaws.com/2010-08-01/distribution/</a></span><span class="Special">'</span> + cf_distribution + <span class="Special">'</span><span class="String">/invalidation</span><span class="Special">'</span>)

req = <span class="Type">Net</span>::<span class="Type">HTTP</span>::<span class="Type">Post</span>.new(uri.path)
req.initialize_http_header({
  <span class="Special">'</span><span class="String">x-amz-date</span><span class="Special">'</span> =&gt; date,
  <span class="Special">'</span><span class="String">Content-Type</span><span class="Special">'</span> =&gt; <span class="Special">'</span><span class="String">text/xml</span><span class="Special">'</span>,
  <span class="Special">'</span><span class="String">Authorization</span><span class="Special">'</span> =&gt; <span class="Special">&quot;</span><span class="String">AWS %s:%s</span><span class="Special">&quot;</span> % [s3_access, <span class="Type">Base64</span>.encode64(digest.digest)]
})

req.body = <span class="Special">&quot;</span><span class="String">&lt;InvalidationBatch&gt;</span><span class="Special">&quot;</span> + paths + <span class="Special">&quot;</span><span class="String">&lt;CallerReference&gt;ref_</span><span class="Special">#{</span><span class="Type">Time</span>.now.utc.to_i<span class="Special">}</span><span class="String">&lt;/CallerReference&gt;&lt;/InvalidationBatch&gt;</span><span class="Special">&quot;</span>

http = <span class="Type">Net</span>::<span class="Type">HTTP</span>.new(uri.host, uri.port)
http.use_ssl = <span class="Boolean">true</span>
http.verify_mode = <span class="Type">OpenSSL</span>::<span class="Type">SSL</span>::<span class="Type">VERIFY_NONE</span>
res = http.request(req)

puts res.code
puts res.body
</pre>

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
