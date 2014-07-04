Using IPython on GStar
=========

After I discovered the wonders of IPython i wanted to use it to analyze some
data, but since all my data is generated on the cluster i wanted to avoid copying
stuff around.

So the idea is, install IPython, and then use `ipython notebook` to do all the
analysis using a browser.

I will follow this convention:
* `g2>` it's the prompt on the head node
* `pc>` it's the prompt on your machine

Let's first check what's available:

    g2> python --version
    Python 2.6.6
    g2> ipython --version
    ipython: command not found

Pretty sad, let's see what else is available

    g2> module avail python
    -------------------------------------------- /usr/local/modules/modulefiles --------------------------------------------
    python/2.7.2(default)

Cool! let's load this and see what changes

    g2> module load python
    g2> $ python --version
    Python 2.7.2
    g2> ipython --version
    0.13.1

Wow it's already there! But wait, IPython.org says that the stable version is 2.1
so we're pretty behinde here, what can we do about it?

Enter `virtualenv`: this is a piece of code that allows you to install a local
version of python and all it's libraries (the `site-packages`) in a place of your
choice. It also installs pip there so you can download and install every missing
package that you might need, let's start by creating a new directory and create
virtualenv there.

    g2> mkdir ipyenv
    g2> virtualenv --system-site-packages ipyenv

> When creating a new `virtualenv` you can choose if you want all the system
> packages available in your environment or not using respectively the ``--system-site-packages`
> or the `--no-site-packages` flag, in the latter case you will end up with only a
> few packages and you will have to install everything you need using pip (numpy, matplotlib, etc..)
>
> But if you work on GStar if you use --no-site-packages seems to have no effect!
> the first time i was expecting a few packages but I got all the system packages
> anyway.
>
> This appens because the `module load python` that we used earlier also sets
>
>     PYTHONPATH=/usr/local/python-2.7.2/lib/python2.7/site-packages
> this does not play well with the virtualenv and if you update a package that is
> already in your system it will hide it behind the system one so a simple
>
>     unset PYTHONPATH
> **this is important because we are going to update IPython and we don't want the system version to
> shadow the updated one**

this should create a new enviroment inside ipyenv:

    g2> ls ipyenv
    bin/ include/ lib/ share/

First of all we need to activate the environment every time we want to use it

    g2> source ipyenv/bin/activate
    (ipyenv)g2>

Now the virtualenv is activated and you can always tell so by looking at the promt!

The next step is to update pip inside this virtualenv (-U means upgrade)

    (ipyenv)g2> pip install -U pip

And with `pip list` you will be able to see which packages are installed.

Now it's time to install ipython (plus some dependencies for the notebook), since it's already in the system packages we need
to use the -U flag again.

    (ipyenv)g2> pip install -U ipython
    (ipyenv)g2> pip install pyzmq
    (ipyenv)g2> pip install tornado

And after a while we will be able to do

    (ipyenv)g2> ipython
    Python 2.7.2 (default, Jul 16 2013, 11:39:13)
    Type "copyright", "credits" or "license" for more information.

    IPython 2.1.0 -- An enhanced Interactive Python.
    ?         -> Introduction and overview of IPython's features.
    %quickref -> Quick reference.
    help      -> Python's own help system.
    object?   -> Details about 'object', use 'object??' for extra details.

    In [1]:

Success!!

Now let's try the notebook

    (ipyenv)g2> ipython notebook
    2014-07-04 15:08:04.057 [NotebookApp] Using existing profile dir: u'/mnt/home/username/.ipython/profile_default'
    2014-07-04 15:08:04.253 [NotebookApp] Using MathJax from CDN: http://cdn.mathjax.org/mathjax/latest/MathJax.js
    2014-07-04 15:08:04.470 [NotebookApp] Serving notebooks from local directory: /mnt/home/username
    2014-07-04 15:08:04.470 [NotebookApp] 0 active kernels
    2014-07-04 15:08:04.470 [NotebookApp] The IPython Notebook is running at: http://localhost:8888/
    2014-07-04 15:08:04.470 [NotebookApp] Use Control-C to stop this server and shut down all kernels (twice to skip confirmation).


What a text browser? This looks so '80s, plus no Javascript so that's not exactly
what we had in mind isn't it? use `q` and then `y` to close it we can see that the notebook it's
still running, if only we could use our browser to connect to it (obviusly you cannot simply
run a server on the head node and expect to be able to connect to it), ssh to the rescue!

To connect to the notebook we need to route our connection through a SSH tunnel
we can easily setup one using this command:

    pc> ssh -L 8888:localhost:8888 username@g2.hpc.swin.edu.au

Basically this tells ssh three thigs:
* Connect to g2.hpc.swin.edu.au
* Keep listening for incoming connection on the local computer, on port 8888
* Forward every connection on the local computer to 'localhost:8888' as if the
connection came from g2.hpc.swin.edu.au (basically to itself)

Now fire up your browser and connect to localhost:8888 , voila'! IPython is ready to
roll!

But at this point I can feel the wrath of our sysadmins as I made all of you run
code on the headnode! I can't live with that, so let's make things work well with our
queue system.

The first thing that comes to mind is running on the interactive nodes (eg gstar001), if you try that
you will notice that you don't know how to connect to the notebook then, because you
cannot ssh directly from your machine to the interactive nodes!

But wait, maybe we can use the tunnel to tell g2 to connect to gstar001
(in this way gstar001 will see a connection from g2 and be happy):

first of all we have to allow for this connection in the notebook config

    (ipyenv)g2> ipython profile create astro

this will create an ipython profile in `/mnt/home/username/.ipython/profile_astro/`,
let's edit `ipython_notebook_config.py` in there, by uncommenting and changing these lines:

    c.NotebookApp.ip = '*'
    c.NotebookApp.open_browser = False

The first is the most important since it allows the notebook to accept incoming connections
from hosts different from `localhost`

Now let's ssh on gstar001, and do the usual commands:

    gstar001> module load python
    gstar001> unset PYTHONPATH
    gstar001> source ipyenv/bin/activate
    (ipyenv)gstar001> ipython notebook --profile=astro

and on your machine

    pc> ssh -L 8888:gstar001:8888 username@g2.hpc.swin.edu.au

This seems neat! But after you point your browser to localhost:8888 and wait a little
you realize that it's not gonna work, connection times out!

That's because the firewall settings don't allow gstar001 to accept incoming connections!
That bugged me for a while, but after a little digging i understood the that's because we
try to connect to it over standard ethernet while we should be using the infiniband network.
But how can I do that? playing with IPython parallel i discovered that it worked flawlessly
and managed to use the right network! So I digged a little bit and I came across this piece of
source code:

    from IPython.utils.localinterfaces import public_ips

That's how the IPython.parallel manages to get the right ip! How can we tell the notebook to do the
same? Well, the notebook config is a python file itself, so we can try to add this lines to
`ipython_notebook_config.py`

    from IPython.utils.localinterfaces import public_ips
    c.NotebookApp.ip = public_ips()[-1]

let's restart the notebook:

    (ipyenv)gstar001> ipython notebook --profile=astro
    2014-07-04 15:46:46.134 [NotebookApp] Using existing profile dir: u'/mnt/home/username/.ipython/profile_astro'
    2014-07-04 15:46:46.396 [NotebookApp] Using MathJax from CDN: http://cdn.mathjax.org/mathjax/latest/MathJax.js
    2014-07-04 15:46:46.476 [NotebookApp] Serving notebooks from local directory: /mnt/home/abibiano
    2014-07-04 15:46:46.476 [NotebookApp] 0 active kernels
    2014-07-04 15:46:46.476 [NotebookApp] The IPython Notebook is running at: http://SOMENEWIP:8888/
    2014-07-04 15:46:46.476 [NotebookApp] Use Control-C to stop this server and shut down all kernels (twice to skip confirmation).

as you can see now "The IPython Notebook is running at: http://SOMENEWIP:8888/" this seems to be working!
 Let's try the tunnel

    pc> ssh -L 8888:SOMENEWIP:8888 username@g2.hpc.swin.edu.au

and firing up your browser to localhost:8888 now works perfectly!

But wait, the sysadmins are not completely appy yet! plus connecting to interactive nodes an writing
commands is a pain, let's wrap all this in a pbs script:

    #!/bin/bash
    #PBS -N myipython
    #PBS -q gstar
    #PBS -l nodes=1:ppn=1
    #PBS -l pmem=1024mb
    #PBS -l walltime=01:00:00:00
    #PBS -k eo
    #PBS -A pXXX_swin

    echo Deploying job to CPUs ...
    echo $PBS_NODEFILE


    module load openmpi/x86_64/gnu/1.6.4
    module load python
    unset PYTHONPATH

    cd /home/username/ipyenv
    echo Working Directory: $PBS_O_WORKDIR


    source bin/activate
    ipython notebook --profile=astro
    echo done


save this as `ipy.pbs` and run it using `qsub ipy.pbs` and while your jobs starts
do a  `tail -f myipython.eXXXXXXX ` this should allow you to see the address to
set up the tunnel and then you are good to go!

I think this is the nicest way to run IPython on the cluster.
Keep an eye on the time, you wouldn't like your ipython notebook to be killed while you
are using it :D
I don't know if this is strictly within the intended use for a cluster especially for long interactive
sessions without much computing power used, as usual be sensible with the resources
and use only what you really need and be sure to kill the job as soon as you finish
your ipython session.
