---
date: "2018-01-18"
title: "Rstudio Server On A Chromebook"
tags: ['chromeOS','Chromebook','crouton','xiwi','RStudio']
categories: ['R']
---

The only post I got around to writing in 2017 was a single messy,
evoloving set of notes on how to set up a Chromebook with R and RStudio.
This post is a reprise of that one as well as post \#1 in my effort to
double my blog output from 2017 (I'm shooting for the stars on that
goal). In this post I will detail how to set up RStudio Server in a
crouton chroot and access it directly via Chrome without having to fire
up a separate X11 window.

Why RStudio Server?
-------------------

I have been succesfully using RStudio Desktop on my Chromebook all year
and it has worked great. But there have been a few issues that have kind
of bothered me. The top four of these are: opening web pages from
RStudio, opening documents (PDF or Word), overall look and feel, and
(the big one) not being able to pull out a source tab into a separate
window. None of these are deal breakers, but they did lead to a bit of a
bumpy workflow at times. I did not have these problems when accessing
RStudio Server in the browser, but I don't want to pay for servers to
host RStudio Server nor do I want to require an internet connection to
work. I figured I could stand up RStudio Server in a chroot, access it
directly in Chrome via `localhost` and have a much more native ChromeOS
feeling experience.

The long and short of it is that it is pretty easy to get set up (if you
are comfortable setting up crouton) and the overall experience is much
improved with web pages and documents just opening up in new tabs as
you'd expect. The biggest improvement is now I can pull out those source
tabs into a stand alone window and take advantage of the real estate of
dual screens!

So enough yammering, how do you do it?

Install crouton and set up your chroot
--------------------------------------

There are a lot of great resources for installing crouton so no need for
me to go over that again. In particular take a look at [Jenny Bryan's
notes](https://github.com/jennybc/operation-chromebook) and the [crouton
repository](https://github.com/dnschneid/crouton)

For RStudio Server we don't need a full desktop so I decided to try this
with a minimal install. I chose core and cli-extra because, frankly, I
didn't know the difference between the two and was just playing it safe.
You can choose to encrypt the chroot or not. I didn't as I was playing
around with auto-starting the chroot when Chrome OS starts ([see here
for more on
that](https://github.com/dnschneid/crouton/wiki/Autostart-crouton-chroot-at-ChromeOS-startup)).
Never got that part of it working right.

*update 2018-01-28:* With just core and cli-extra you are missing a few
things. Most notably for me, sound was not working correctly. And well,
I need [Rasmus Bååth's beepr
package](https://cran.r-project.org/package=beepr). Adding the
`extension` target fixed this.

*update 2018-01-30:* While extension fixed the audio issue, I was still
having problems with RStudio Server and R Session getting
discombobulated. I tried a full desktop install and that appears to be
working much better. So the idea of using the minimal install wasn't so
great as some needed tooling was obviously missing. The rest of this
post has been updated to reflect this.

Anyway, here is what I used to get my chroot set up.

    sudo sh ~/Downloads/crouton -t xfce,extension -n rstudio

Once that finishes (it takes a while), you can hop into the chroot with

    sudo enter-chroot

Install R
---------

With a shiny new chroot started, we can now start all of our installs.
First one is R.

    sudo echo "deb http://cran.rstudio.com/bin/linux/ubuntu xenial/" | sudo tee -a /etc/apt/sources.list
    gpg --keyserver keyserver.ubuntu.com --recv-key E084DAB9
    gpg -a --export E084DAB9 | sudo apt-key add -
    sudo apt-get update
    sudo apt-get install r-base r-base-dev

Install RStudio Server
----------------------

Next, we need to install RStudio Server. This grabs the latest and
greatest version as of January 2018. I choose to delete the `.deb` just
to keep the footprint of this thing to a minimum.

    sudo apt-get install gdebi-core
    wget https://download2.rstudio.org/rstudio-server-1.1.419-amd64.deb
    sudo gdebi rstudio-server-1.1.419-amd64.deb
    rm rstudio-server-1.1.419-amd64.deb

As an aside, I got distracted when I started working on this section. I
wanted to be able to download the current version of RStudio Server
without actually knowing what that version was. Luckily, RStudio lists
those as `XML` at <https://downloads2.rstudio.org>. Unluckily, I know
next to nothing about working with `XML` so this code certainly reflects
that as I convert to string and parse that for the current version. If
you are interested, here's the function that gets the current version of
RStudio Server. Also if you have thoughts on a more direct root (seems
like it could be done within the `XML` itself), let me know in the
comments.

    get_rstudio_server <- function(arch = c("amd64","i386","i686","x86_64"),
                                   file_type = c(".deb",".rpm"),
                                   pro = FALSE, get = TRUE){
      arch <- match.arg(arch)
      file_type <- match.arg(file_type)
      url <- 'https://download2.rstudio.org'
      dat <- xml2::read_xml(url)
      dat_txt <- xml2::xml_text(xml2::xml_children(dat)) 
      current <- dat_txt[stringr::str_detect(dat_txt,"current")]
      current_date <- stringr::str_sub(current,12,21)
      deb_files <- dat_txt[stringr::str_detect(dat_txt,current_date)]
      deb_files <- deb_files[stringr::str_detect(deb_files,paste0(arch,file_type))]
      if(length(deb_files) == 0){
        stop("Not a valid arch and file_type combination")
      }
      if(pro){
        deb_file <- deb_files[stringr::str_detect(deb_files, "pro")]
      } else {
        deb_file <- deb_files[!stringr::str_detect(deb_files, "pro")]
      }
      deb_file <- stringr::str_sub(deb_file,1,
                                   stringr::str_locate(deb_file,current_date)[1]-1)
      deb_file <- deb_file[1]
      if(get){
        httr::GET(paste0(url,"/",deb_file),httr::write_disk(deb_file, overwrite = TRUE),
                  httr::progress())
      }
      deb_file
    }

Open up firewall
----------------

Since this is RStudio server, we will be accessing it via Chrome and
will need to make sure the requests get through the chroot's firewall.
We need to open that up to do so. I include the `nano` install because
it is a little easier to work with than `vim`.

    sudo apt-get install nano
    sudo nano /etc/rc.local

Then add `/sbin/iptables -I INPUT -p tcp --dport 8787 -j` to the end of
the file. While I was editing the file, I also added
`/sbin/iptables -I INPUT -p tcp --dport 4321 -j` to open up the port for
Hugo. I'm switching to `blogdown` for my website and wanted to be able
to easily get at the preview versions.

*update 2018-01-30:* Actually forgot to do this with my test of the xfce
desktop and I had no trouble access the server. So, apparently not
needed. I'm keep the instructions here mostly so I have access to them
if other things (i.e. hugo) do require them.

Get RStudio server running and access it
----------------------------------------

Now you need to start up the server.

    sudo rstudio-server start

Then in your browser you can access your RStudio Server with
`localhost:8787`

Draw the rest of f\*\*\*ing owl
-------------------------------

Apologies if you have seen the ["how to draw an owl"
meme](http://knowyourmeme.com/memes/how-to-draw-an-owl). It feels
appropriate at this point. What follows are all the things I had to add
to get up and running with my standard set-up.

First, let's change the locale and add some needed tools

    sudo locale-gen "en_US.UTF-8"
    sudo dpkg-reconfigure locales
    sudo apt-get install software-properties-common libxslt1-dev libcurl4-openssl-dev libssl-dev libssh2-1-dev

Next I needed to add the Ubuntu GIS repositories so I can get the up to
date versions of [GDAL](), [GEOS](), etc.

    sudo add-apt-repository ppa:ubuntugis/ubuntugis-unstable
    sudo apt-get update

And add my GISy libraries

    sudo apt-get install libgdal-dev libproj-dev libudunits2-dev

Then we can add LaTeX

    sudo apt-get install texlive texlive-latex-extra texlive-pictures

In my experience this is close to the minimal LaTeX install that will
work with pandoc, R Markdown, etc.

Lastly, some version control!

    sudo apt-get install git

And then I configure git.

    git config --global user.name "Your Name"
    git config --global user.email "your.email@email.com"
    git config --global credential.helper 'cache --timeout=28800'

Day to day use
--------------

There is a way to have your chroot automatically boot with Chrome OS and
RStudio Server is supposed to start on boot. For some reason, the server
wasn't kicking off when I started up the chroot. I didn't track this
down so don't know if it will work. Seems like with some hacking it
should.

So given that I couldn't get everything starting automagicaly, the
simplest solution I came up with is to:

-   start up my chromebook
-   open crosh - ctrl-alt-t
-   type `shell`
-   type `sudo enter-chroot`
-   type `sudo rstudio-server start`
-   type `localhost:8787` into a broswer tab.

Another way to go with this is set up a couple of aliases in
`~/.bashrc`. You can do this with:

-   at your crosh shell: `vi ~/.bashrc`
-   add this to the file: `alias rstudio="sudo enter-chroot -n rstudio"`
-   now type in the shell `. ~/.bashrc`.  
-   type `rstudio` in the shell. Once you enter your password, you will
    be in your chroot.
-   enter some aliases for starting the server
-   type `nano ~./bashrc`
-   add this: `alias rstudio="sudo rstudio-server start"`
-   type rstudio, enter password.
-   type `localhost:8787` into a browser tab.

The first time I get RStudio open I want to install all my usual
suspects. With this set up I had no issues installing the following

    pkgs <- c("devtools","tidyverse","sf","raster","mapview","sp","rgdal","rgeos","roxygen2", "blogdown")

    lapply(pkgs, install.packages)

I then install my own packages, just so I have them handy

One little hiccup
-----------------

The shutdown process is not super smooth on this (only problem I have
identified so far). When your chromebook sleeps, or you shut down the
chroot, etc. The R session and RStudio get confused. A scary error will
pop-up when you try to get backin in and at this point, I can no longer
switch between projects. When you select a new project it looks like it
is switching, but ends keeping you at your root without a project. If
you don't use projects then this might not be an issue (but you do risk
the [arsonistic ire of Jenny
Bryan](https://www.tidyverse.org/articles/2017/12/workflow-vs-script/)).
If you do use projects this is a problem. I did find a very simple
workaround for when this happens. All you need to do is start a new
session. This can be done with the little red power button icon in the
upper right corner of the window or with <File:Quit> Session.

*update 2018-01-30:* Using the xfce desktop install seems to have fixed
this issue. No need to unnecessarily start new session now!

So, there you have it. RStudio on a chromebook via RStudio Server
running in a chroot! I am now very happy with the set-up and fully
expect to stick with the Chromebook for some time to come.
