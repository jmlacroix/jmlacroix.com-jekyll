---
layout: post
title: Replacing a Development VPS with Linux on OSX
desc: Using QEMU to run a lightweight background Linux VM for development.
---

As much as I love my Mac, I'm sometimes having uncontrollable urges to install Linuxy stuff on it. Sometimes it works great (thanks mostly to [homebrew][brew]), but at other times it's flaky or impossible.

So I've been looking for a Linux VPS, to have a Linux machine handy when I want to play around and try stuff. But, since I'm using [QEMU][qemu] (a lightweight virtualization tool) a lot at work, I tried using it as a background Linux virtual machine that's always available.

I now have a FREE, always-on (using negligible CPU and RAM when idle) Linux machine that is available whenever I need it, wherever I am. It's also quite useful to test tools so I don't mess with my *stable* environment: it's too easy to break the world when installing lots of stuff.


Installation
------------

Download an ISO version of the distribution you want to install (in my case, the excellent [Arch Linux][arch]) and fire up your favorite terminal.

Install QEMU <sup id="fn1">[1]</sup>:

    brew install qemu


Create a Qcow2 image, that can expand up to 10GB:

    qemu-img create -f qcow2 archlinux.qcow2 10G

Launch QEMU with your image attached as main hard drive and your ISO as cdrom:

    qemu-system-x86_64 -m 512 -hda archlinux.qcow2 -cdrom archlinux.iso

Before shutting it down, make sure you enable the OpenSSH sever so you can log in later.

To access the VM via SSH from the local machine, boot it with a network redirection:

    qemu-system-x86_64 -m 512 -hda archlinux.qcow2 -redir tcp:2222::22

This redirects all traffic from localhost port 2222 to VM port 22.

When it's done booting, try to connect to via SSH with the following command:

    ssh username@localhost --port 2222

Once you get this working, shut it down.


Usage
-----

There's many ways to start the machine, I use a bash function in my `~/.bashrc` file to launch it when I need it:

    function linux {
        qemu-system-x86_64 -m 512 -hda archlinux.qcow2 -redir tcp:2222::22 \
                           -nographic -daemonize
    }

Notice the `-nographic` and `-daemonize` options: they ensure the machine is started as a background process.

For faster connection, add these settings in your `~/.ssh/config` file:

    Host linux
        User     username
        Hostname localhost
        Port     2222

You can now connect and copy files to your new home easily with

    ssh linux

and

    scp file.txt linux:

For maximum fun, setup a shared folder on your local machine that your VM can access for easier file sharing. It's also a good idea to make a backup copy of your stable image so that you can replace a broken VM by a clean new one.

Have fun playing with your new development environment.

<div class="footnotes">
<ol>
<li id="ffn1"><p>
There seems to be a problem with the BIOS in daemon mode in QEMU 0.15 on OSX, so make sure you install version 0.14.1.<br/><pre>brew install https://raw.github.com/mxcl/homebrew/bf2dd2bea04daf78a98888cf20fdf438fb777112/\
Library/Formula/qemu.rb</pre> <a href="#fn1" title="Jump back to footnote 1 in the text.">&#8617;</a></p></li>
</ol>
</div>

[brew]: http://mxcl.github.com/homebrew/
[qemu]: http://wiki.qemu.org/Main_Page
[arch]: http://archlinux.org

[1]: #ffn1