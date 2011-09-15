---
layout: post
title: Exporting VIM Syntax Highlight
desc: Using
---

I've been looking for a tool to replace the GitHub Gists I'm using to highlight code on these pages. Gists are great, but I wanted something that doesn't rely on JavaScript (for SEO, accessibility and performance reasons).

While taking a look at 3rd party libs, I flashed: why not use the output of my favorite editor, VIM? It has powerful syntax highlighting and a lot of syntax files. I can even use color schemes and some of VIM's settings, such as line numbering, to make my life easier (and it reads my `vimrc`!).

So instead of using simple tools, I went for the complicated solution. I made two script to help extracting HTML-formatted syntax and its associated CSS styles. Note that these scripts outputs their results on `stdout`, so they won't alter your files.


Converting to HTML
------------------

[`highlight.sh`][hi] outputs HTML-formatted syntax highlight of a file:

<pre>
<span class="Comment">#/bin/bash</span>

<span class="Identifier">OUTFILE</span>=<span class="PreProc">$(</span><span class="Special">mktemp -t highlight</span><span class="PreProc">)</span>

<span class="Identifier">INFILE</span>=<span class="PreProc">$1</span>
<span class="Identifier">PARAM</span>=<span class="Statement">&quot;</span><span class="String">set nonumber</span><span class="Statement">&quot;</span>

<span class="Statement">if </span><span class="Statement">[</span> <span class="Statement">-z</span> <span class="Statement">&quot;</span><span class="PreProc">$INFILE</span><span class="Statement">&quot;</span> <span class="Statement">]</span>
<span class="Statement">then</span>
    <span class="Statement">echo</span><span class="String"> </span><span class="Statement">&quot;</span><span class="String">usage: </span><span class="PreProc">$0</span><span class="String"> source.file</span><span class="Statement">&quot;</span>
    <span class="Statement">exit</span>
<span class="Statement">fi</span>

vim +<span class="Statement">&quot;</span><span class="PreProc">$PARAM</span><span class="Statement">&quot;</span> <span class="Statement">\</span>
    +<span class="Statement">'</span><span class="String">let html_use_css=1</span><span class="Statement">'</span> <span class="Statement">\</span>
    +<span class="Statement">'</span><span class="String">TOhtml</span><span class="Statement">'</span> <span class="Statement">\</span>
    +<span class="Statement">'</span><span class="String">/&lt;pre&gt;/,/&lt;\/pre&gt;/d a</span><span class="Statement">'</span> <span class="Statement">\</span>
    +<span class="Statement">'</span><span class="String">g/./d</span><span class="Statement">'</span> <span class="Statement">\</span>
    +<span class="Statement">'</span><span class="String">1pu! a</span><span class="Statement">'</span> <span class="Statement">\</span>
    +<span class="Statement">'</span><span class="String">$d</span><span class="Statement">'</span> <span class="Statement">\</span>
    +<span class="Statement">&quot;</span><span class="String">wq! </span><span class="PreProc">$OUTFILE</span><span class="Statement">&quot;</span> <span class="Statement">\</span>
    +<span class="Statement">'</span><span class="String">q!</span><span class="Statement">'</span> <span class="PreProc">$INFILE</span> &amp;<span class="Statement">&gt;</span>/dev/null

cat <span class="PreProc">$OUTFILE</span> &amp;&amp; <span class="Statement">rm</span> <span class="PreProc">$OUTFILE</span>
</pre>

Try the script on itself:

    $ bash highlight.sh highlight.sh


Extracting CSS
--------------

[`syntax.sh`][sy] loads VIM's syntax test and outputs its generated CSS styles:

<pre>
<span class="Comment">#!/bin/bash</span>

<span class="Identifier">OUTFILE</span>=<span class="PreProc">$(</span><span class="Special">mktemp -t syntax</span><span class="PreProc">)</span>

vim +<span class="Statement">'</span><span class="String">so $VIMRUNTIME/syntax/hitest.vim</span><span class="Statement">'</span> <span class="Statement">\</span>
    +<span class="Statement">'</span><span class="String">let html_use_css=1</span><span class="Statement">'</span> <span class="Statement">\</span>
    +<span class="Statement">'</span><span class="String">TOhtml</span><span class="Statement">'</span> <span class="Statement">\</span>
    +<span class="Statement">'</span><span class="String">v/^\./d</span><span class="Statement">'</span> <span class="Statement">\</span>
    +<span class="Statement">&quot;</span><span class="String">wq! </span><span class="PreProc">$OUTFILE</span><span class="Statement">&quot;</span> <span class="Statement">\</span>
    +<span class="Statement">'</span><span class="String">q!</span><span class="Statement">'</span> &amp;<span class="Statement">&gt;</span>/dev/null

cat <span class="PreProc">$OUTFILE</span> &amp;&amp; <span class="Statement">rm</span> <span class="PreProc">$OUTFILE</span>
</pre>

Extract current vim colors as CSS:

    $ bash syntax.sh


Further Integration
-------------------

Currently, it's not the best way to integrate code into my posts. It's not automated and I need to manually insert the HTML. I've made an `awk` script to help with this task. I'll post about it when it's integrated into my process.


[hi]: https://gist.github.com/662aa4e8aea8d4b4070b
[sy]: https://gist.github.com/1ef9e5f8287cf6320d19
