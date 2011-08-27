---
layout: post
title: DRY your ssh config with rubbissh
desc: rubbissh is a ruby script that generates a ssh config file from a YAML
      definition.
---

I really hate to repeat myself, and that happens way too much when managing
my ever-growing `~/.ssh/config` file. I decided to get my hands dirty and
created a tool to handle all the repeating.

## How It Works ##

Define your ssh rules in a YAML formatted `config.yml` file:

	*:
	  server_alive_interval: 30
	  server_alive_count_max: 120

	dev-:
	  *:
	    user: bobby
	    port: 1221

	  website: my-web-host.com
	  database:
	    host_name: my-db-host.com
	    identity_file: ~/ident.key

Use `rubbissh` to generate a classic ssh config file:

	Host *
		ServerAliveCountMax	120
		ServerAliveInterval	30
	Host dev-*
		User	bobby
		Port	1221
	Host dev-database
		HostName	my-db-host.com
		IdentityFile	~/ident.key
	Host dev-website
		HostName	my-web-host.com

## More Details ##

- Using the `*` wildcard at any level will define default keywords for all
machines in the group.
- Using a `-` at the end of a machine name creates a server group.
- The default vaue of a `server`: `string` match is its host name.
- You can nest as much definitions as you want.

You can get more information on installation, fork the sources or report
problems [on GitHub](http://github.com/jmlacroix/rubbissh).
