---
author: Vanessasaurus
blog_subtitle: dinosaurs, programming, and parsnips
blog_title: VanessaSaurus
blog_url: https://vsoch.github.io/
category: vsoch
date: '2022-08-04 13:30:00'
layout: post
original_url: https://vsoch.github.io/2022/hpc-apps/
title: Tunel- Apps for HPC
---

<p>A few months ago I was talking about <a href="https://vsoch.github.io/2022/ssh-tunnels/" target="_blank">ssh tunnels</a>. The reason was because I was looking for a solution to deploy apps (like a Jupyter notebook) onto HPC.
After an adventure I got it working, and it came down a relatively simple set of commands that I needed to just <a href="https://github.com/tunel-apps/tunel/blob/main/tunel/ssh/commands.py">write into my app logic</a> and forget about.
The reason for this was working on my new personal project, <a href="https://tunel-apps.github.io/tunel/" target="_blank">tunel</a>.</p>

<blockquote>
  <p>Tunel is named for what it does. “Tunel” is an elegant derivation of “tunnel” and will do exactly that - create a tunnel between your local workstation and an HPC cluster.</p>
</blockquote>

<div style="padding: 20px;">
 <img src="https://vsoch.github.io/assets/images/posts/tunel/tunel-docs.png" />
</div>

<p>In short, tunel will provide a collection of “apps” that are easy to deploy to HPC. There are concepts called launchers, and examples are singularity, slurm, or htcondor. And we can add more! It’s the job of a launcher to take a an app recipe (a definition in yaml plus helper scripts that can be customized on the fly by the user) and get it running, whatever that means (run a job? a container? monitor something? something else?). For the most part, most apps that I’ve developing have web interfaces, as they have historically been the most challenging thing to get easily working on HPC. As a quick example, to run a jupyter notebook via Singularity on my login node, after I install tunel and have my ssh connection defined as “osg” I can do:</p>

<div class="language-bash highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="nv">$ </span>tunel run-app osg singularity/socket/jupyter <span class="nt">--jupyterlab</span><span class="o">=</span><span class="nb">true</span> 
</code></pre></div></div>

<p>The name “singularity/socket/jupyter” is the unique identifier (and path) to the recipe and config, and I can provide custom arguments as shown above. And although this is the “singularity” launcher, we can do the same kind of interaction with a slurm launcher, going one level deeper to run the notebook on a node after we submit a job!
And in my typical way of doing things, I have automation that generates a table and documentation for each of these apps. <a href="https://tunel-apps.github.io/tunel/_static/apps/" target="_blank">Check them out here!</a>.</p>

<div style="padding: 20px;">
 <img src="https://vsoch.github.io/assets/images/posts/tunel/table.png" />
</div>

<p>I’m mostly working on singularity an HTCondor apps at the moment because I use the open science grid (OSG) for development, as this is a personal project. Thanks to <a href="https://twitter.com/westbynoreaster" target="_blank">Matthew West</a> for showing me OSG - I was pretty handicapped to develop before finding it!</p>

<h2 id="django-template-with-a-socket">Django template with a socket?</h2>

<p>This kind of framework can be powerful if I develop a bunch of custom apps, but it’s much more powerful if I can enable YOU to easily do that too! Thus, I knew one of the first tasks I wanted to do is create a template, likely in each of Flask, Django, and FastAPI, that would plug immediately into Tunel. And while I have much work left to do, last night and this evening I figured out a technical issue that is going to empower us to make so many cool things and I wanted to share! Let’s talk about the problem, what I tried, and what ultimately worked.</p>

<h3 id="traditional-setup-with-uwsgi-and-nginx">Traditional Setup with uwsgi and nginx</h3>

<p>If you look at a family of Python + web interface apps, you’ll find this <a href="https://uwsgi-docs.readthedocs.io/en/latest/" target="_blank">uwsgi</a> guy in the middle (I don’t know the correct pronunciation but I say YOU-SKI). It’s a fairly rich tool, but in layman’s terms I think of it as a middleman between Python and a traditional web server. But actually, you don’t technically need the web server - and this is where things start to get interesting. For a traditonal setup, you might find a nginx (a web server) configuration file that <a href="https://github.com/tunel-apps/tunel-django/blob/main/scripts/nginx/nginx.conf" target="_blank">looks like this</a>.</p>

<div class="language-yaml highlighter-rouge"><div class="highlight"><pre class="highlight"><code>
<span class="c1"># the upstream component nginx needs to connect to</span>
<span class="s">upstream django {</span>
    <span class="s">server unix:///tmp/tunel-django.sock;</span>
<span class="err">}</span>

<span class="c1"># configuration of the server</span>
<span class="s">server {</span>
    <span class="s"># the port your site will be served on</span>
    <span class="s">listen      8000;</span>
    <span class="s">charset     utf-8;</span>
    <span class="s">server_name           localhost;</span>

    <span class="s">client_max_body_size 10024M;</span>
    <span class="s">client_body_buffer_size 10024M;</span>
    <span class="s">client_body_timeout 120;</span>

    <span class="s">...</span>
    <span class="s">location ~* \.(php|aspx|myadmin|asp)$ {</span>
      <span class="s">deny all;</span>
    <span class="s">}</span>

    <span class="s">location /static/ {</span>
        <span class="s">autoindex on;</span>
        <span class="s">alias /var/www/static/;</span>
    <span class="s">}</span>

    <span class="s"># Finally, send all non-media requests to the Django server.</span>
    <span class="s">location / {</span>
        <span class="s">uwsgi_pass  django;</span>
        <span class="s">uwsgi_max_temp_file_size 10024m;</span>
        <span class="s">include /code/scripts/nginx/uwsgi_params.par;</span>
    <span class="s">}</span>
<span class="err">}</span>
</code></pre></div></div>

<p>I’ve made a lot of web apps, and whether I use docker-compose with separate containers or a single one, I usually have to write a nginx configuration. The above gets started in the container entrypoint with my app calling uwsgi, and defining that same socket:</p>

<div class="language-bash highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="nv">$ </span>uwsgi <span class="nt">--socket</span><span class="o">=</span><span class="k">${</span><span class="nv">socket</span><span class="k">}</span> /code/scripts/uwsgi.ini
</code></pre></div></div>

<p>And of course things happen before that, but that’s the main last line. The uwsgi.ini is a configuration file
that makes it easier to define settings.</p>

<div class="language-plaintext highlighter-rouge"><div class="highlight"><pre class="highlight"><code>[uwsgi]
master = true
processes = 4
threads = 4
py-autoreload = 1
#socket = :3031
chdir = /code/
post-buffering = true
log-date = true
max-requests = 5000
http-timeout = 3600000
socket-timeout = 120
chmod-socket = 666
wsgi-file = tuneldjango/wsgi.py
ignore-sigpipe = true
ignore-write-errors = true
disable-write-exception = true
buffer-size=32768
</code></pre></div></div>

<p>Without going into huge detail, the above says that the app that I wrote (in Python) is listening on that socket, so requests to the web server will either be directed to some static file, filtered out, or sent to our application. And we typically want to use nginx because it’s really good at serving static files and handling traffic.</p>

<p>But now let’s step back. If you look under the server in the config above, you’ll notice we are serving
content on port 8000. This is why I can open the browser to localhost and that port and see my application.
But as we know with headless HPC, there are no ports. I can’t use this. So this was my first predicament, last night. I had created this application and it ran locally, but I needed to somehow get the entire thing routed through a tunneled socket to take a next step.</p>

<h3 id="uwsgi-only">Uwsgi Only?</h3>

<p>I’ll skip over the many hours of things that I tried and failed. I really liked having nginx so I first wanted to somehow send it to the user via a socket, but that never worked. I had an idea to just map the original socket and then have a second container on the host for nginx, but I decided that was too complex. What would up working is realizing that uwsgi can serve http directly, and that came down to a single addition to its config:</p>

<div class="language-plaintext highlighter-rouge"><div class="highlight"><pre class="highlight"><code>listen=200
protocol=http
</code></pre></div></div>

<p>Once I did that, I tried the same technique to map the socket being written to directly to a port via the ssh tunnel, and <em>boum</em> I saw a page! But it was really ugly, because it had no style. This is where I was like OHNO I need nginx for static. But then I found <a href="https://uwsgi-docs.readthedocs.io/en/latest/StaticFiles.html" target="_blank">this page</a> and it was a message from the heavens - I could define the same static and media URls using uwsgi directly! That looked like this:</p>

<div class="language-bash highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="nv">$ </span>uwsgi <span class="nt">--socket</span><span class="o">=</span><span class="k">${</span><span class="nv">socket</span><span class="k">}</span> <span class="nt">--static-map</span> /static<span class="o">=</span>/code/static /code/scripts/uwsgi-standalone.ini
</code></pre></div></div>
<p>At this point I held my breath, re-ran my app, and wow!</p>

<div style="padding: 20px;">
 <img src="https://vsoch.github.io/assets/images/posts/tunel/home.png" />
</div>

<p>There it was - my entire app being served by a container running on a remote machine, only accessible to me through a physical socket. And guess what? I added a file browser, and it even worked to upload a dinosaur picture!</p>

<div style="padding: 20px;">
 <img src="https://vsoch.github.io/assets/images/posts/tunel/browser.png" />
</div>

<p>Here is the entire page for the app - you can see there are many flags you can add and customize to interact.</p>

<div style="padding: 20px;">
 <img src="https://vsoch.github.io/assets/images/posts/tunel/app.png" />
</div>

<p>While it’s only accessible to you and there isn’t need for any kind of login, I did add the default username/password login to Django, and require it for logging in to the file browser. Of course I will eventually need this to be more formally security audited, but at least I don’t have anything interesting on my OSG home to be worried about. And is using just uwsgi a performance issue? I think probably not since the expected use case is only once person.</p>

<h3 id="a-future-for-apps">A Future for Apps</h3>

<p>This is just the beginning - my plan is to put together a list of use cases for a GUI on a cluster, and then just package them into the core template apps for the developer user to easilyc customize. I have big plans for working on this, and honestly I’m so excited that I find I’m staying up way too late and just egging for the work day to end so I can continue. This idea is so powerful, because it’s using existing technologies to deploy containerized apps on HPC, where you don’t need any special permission. Just to show y’all, here is what it looks like to launch my app template:</p>

<div class="language-bash highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="nv">$ </span>tunel run-app osg singularity/socket/tunel-django <span class="nt">--tag</span><span class="o">=</span>dev <span class="nt">--pull</span>
</code></pre></div></div>

<p>I added the pull flag and a custom tag because I am actively developing, and my workflow is to quickly rebuild, push, and then run that command. That then shows me the ssh tunnel command that will immediately connect me to my app on a port in my browser.</p>

<div class="language-bash highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="nv">$ </span>ssh <span class="nt">-NT</span> <span class="nt">-L</span> 7789:/../tunel/singularity/singularity/socket/tunel-django/singularity-socket-tunel-django.sock sochat1@osg
</code></pre></div></div>
<p>And that’s seriously it. You as the developer user are empowered to make and deploy apps, and they have interfaces, and you don’t need to do something silly like open a port or actually deploy a web server. It’s so stupidly easy - I’m looking around at all these complex web app setups that people have made for HPC over the years and I wonder why they aren’t doing something simpler. Maybe it’s just a space of development that people gave up on, or there are some security things I’m missing. Either way, I’m going to charge forward working on this! It’s too simple, and the idea is to beautiful to do anything else by this point.</p>