---
layout: post
title: Emacs, Git, Business & Pleasure
tags: ['emacs', 'git']
category: workflow
comments: true
---
Emacs is an amazing editor; it can be simple, complex, overwhelming, and powerful.

##The Motivation
Every [emacs](http://www.gnu.org/software/emacs/) user I know (_all both of them_)
has at one point lost their emacs configuration.  After 'misplacing' my config (_for about the 3rd
time_) I decided to do something about it.   My solution: use [Dropbox](http://dropbox.com).

The Dropbox solution was simple and it seemed to do exactly what I wanted.
For months I thought I had a workable solution;  Two dropbox accounts (_**work** and **home**_) with folder sharing 
between the two emacs config folders.  Then came the Monday morning a couple weeks ago; I showed up
to work and realized the fancy modifications I made to my config over the weekend were missing.
But not just the changes for that weekend,  it had been out of sync for a while.  The Dropbox folder
sharing had failed me (_and I still don't know exactly when or why_).

My outdated configs made me pull the trigger on something I've wanted to do for a while; switch to [Git](http://git-scm.com/).
It's the perfect solution for managing my emacs config.


##The Structure
Currently, my emacs config folders exist in `~/.emacs.git/` in OS X, and in `%USERPROFILE%/.emacs.git/` on Windows. (_I
might change this in the near future to place everything within the `~/.emacs.d` folder_)

Lets take a look at the directory structure I have in my `~/.emacs.git` folder (_it's a pretty simple structure_).

{% highlight bash %}
~/.emacs.git $ ls -la
drwxr-xr-x   9 jfilsinger  staff   306 25 Dec 23:16 .
drwxr-xr-x+ 54 jfilsinger  staff  1836 28 Dec 05:46 ..
drwxr-xr-x  16 jfilsinger  staff   544 29 Dec 02:47 .git
-rw-r--r--   1 jfilsinger  staff   114 23 Dec 04:34 .gitignore
-rw-r--r--   1 jfilsinger  staff  1048 25 Dec 22:54 .gitmodules
-rw-r--r--   1 jfilsinger  staff  8601 25 Dec 22:57 common-init.el
drwxr-xr-x  11 jfilsinger  staff   374 25 Dec 22:54 submodules
{% endhighlight %}

I made the decision early-on to take advantage of git submodules; every emacs extension I use
has a git repository somewhere (_so why not take advantage_).   Submodules also allow me to make my config
public by moving work-related things into their own private submodule (_leaving me free to make everything else public_).

The `common-init.el` is loaded from the `~/.emacs` file (with an extra variable, `emacs-sync-path`, defined before loading the `common-init.el`.

Here is my `~/.emacs`:
{% highlight cl %}
(defvar emacs-sync-path (if (or (eq system-type 'cygwin)
			     (eq system-type 'gnu/linux)
			     (eq system-type 'linux)
			     (eq system-type 'darwin))
			  (concat (getenv "HOME") "/.emacs.git/")
			  (concat (getenv "USERPROFILE") "/.emacs.git/"))
  "emacs sync directory")
(add-to-list 'load-path emacs-sync-path)
(load "common-init.el")
{% endhighlight %}

Take not of the `emacs-sync-path` variable, it gets used within the `common-init.el`.

##The Submodules
The power of my emacs setup comes from the hard work of others,  lets take a look at how I harness their genius.

I will use [`markdown-mode.el`](http://jblevins.org/projects/markdown-mode/) as an example of how I use the submodules (_The steps are roughly the same for all submodules_).

### Initializing the Submodule
From within the `~/.emacs.git` folder I ran the following commands:
{% highlight bash %}
git submodule add git://jblevins.org/git/markdown-mode.git submodules/markdown-mode
git submodule update --init
git add submodules/markdown-mode
{% endhighlight %}
"**`git submodule add git://jblevins.org/git/markdown-mode.git submodules/markdown-mode`**" - adds the `markdown-mode` submodule to the `~/.emacs.git/submodules/` folder.

"**`git submodule update --init`**" - Initialize the submodule and pull the latest `HEAD` from the remote server.

"**`git add submodules/markdown-mode`**" - The third command lets the git superproject know which SHA the submodule is using. 

* _Notice that the 3rd line does **NOT** have '**/**' at the end.  If the path we are adding ends with '**/**', git thinks think we are adding the directory [and all its contents] and not the submodule_

### Using the Submodule
Using the `emacs-sync-path` variable that was defined within the `~/.emacs` file, I add the `markdown-mode` submodule
path to the emacs `load-path` within my `common-init.el` file.
{% highlight cl %}
(add-to-list 'load-path (concat emacs-sync-path "/submodules/markdown-mode"))
(setq auto-mode-alist 
	(cons '("\\.\\(md\\|markdown\\)$" . markdown-mode) auto-mode-alist))
(autoload 'markdown-mode "markdown-mode" "Markdown editing mode." t)
(add-hook 'markdown-mode-hook 'turn-on-font-lock)
(add-hook 'markdown-mode-hook 'hs-minor-mode)
{% endhighlight %}

### Updating Submodules
Using submodules allows me to easily update the emacs extensions I use with a little bit of shell magic.

{% highlight bash %}
git submodule foreach git checkout master
git submodule foreach git pull
{% endhighlight %}

"**`git submodule foreach git checkout master`**" - loops through all of the submodules the git superproject knows about and tells each of those submodules
to checkout their **`master`** branch. (_we need to do this because when we first add a submodule we get a floating head, submodules
don't initialize to a branch_)

"**`git submodule foreach git pull`**" - loops through all of the submodules and pulls in the latest changes from their remote repository (_I pull because I don't make changes to the submodules_)

## The Rest of the Config
For the most part my `common-init.el` emacs config just loads extensions from submodules and modifying some settings.
If you're interested in seeing it,  you can find it [here](https://github.com/filsinger/emacs-config/blob/master/common-init.el)

## The Work/Life Balance 
Like I mentioned earlier, one of the submodules is a private repository containing all of my work related emacs config options.

Here is a summary of what it contains:

* **snippets** (_I use [yasnippet](https://github.com/capitaomorte/yasnippet)_)
* **modes** (_A mode for parsing TTY output as well as a mode for the scripting language we use_)
* **auto-completion dictionaries** (_For the scripting language we use at work_)
* **functions** (_for calling external work tools/scripts from within emacs_)

## Wrapping It Up
Overall,  I'm content with my emacs setup.  Using submodules allows me to take advantage of the various
submodules on all the computers I use with minimal maintenance. I'm able to update all of the submodules
with just a few shell commands which leaves me with just one file (_the `common-init.el`_) to maintain.

You can find my [emacs config repository](https://github.com/filsinger/emacs-config) on [GitHub](https://github.com).



