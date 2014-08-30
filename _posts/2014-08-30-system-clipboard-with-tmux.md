---
layout: post
title: System Clipboard With tmux
tags: ['tmux', 'osx', 'cygwin', 'linux', 'emacs']
category: workflow
comments: false
meta-description: "Jason Filsinger discusses how he uses tmux with the system clipboard on various platforms."
license: "cc-by-nc-sa"
---
Using [tmux](http://tmux.sourceforge.net) is huge win while working in a terminal, help make
it better by adding support for your system clipboard.


## The Problem
Although [tmux](http://tmux.sourceforge.net) does provide a built-in clipboard, the contents
of this clipboard are not synchronized with the system clipboard. Lucky for us, this problem
can be solved (and in a way that will allow us to share our tmux config across multiple platforms).

## The Solution
Thanks to the way [tmux](http://tmux.sourceforge.net) is designed we are able to script it via
the shell.  If we bind shell commands to the default copy/paste keychords in `~/.tmux.conf` we
can take full control of what happens when we copy and paste.

Here is an example of what what you would put in your `~/.tmux.conf` :
{% highlight conf %}
# copy:
bind-key -n -t emacs-copy M-w copy-pipe "my_copy_command"
# paste:
bind ] run "my_paste_command | tmux load-buffer - ; tmux paste-buffer"
{% endhighlight %}

* **`copy-pipe`** : runs the command you specify and passes the tmux clipboard buffer via stdin.
* **`run`** : executes a shell command.
* **`load-buffer`** : load text into the [tmux](http://tmux.sourceforge.net) buffer *(will load via stdin when followed by a single '`-`')*.
* **`paste-buffer`** : paste the [tmux](http://tmux.sourceforge.net) buffer into the active tmux pane.

By mixing the above example with the `if_shell` command we can specify different behavior for
individual platforms:
> **`if_shell`** `'`*`<shell_command>`*`' '`*`<tmux_success_command>`*`' '`*`<tmux_failure_command>`*`'`

Example:
{% highlight conf %}
# Mac OS X
if-shell "uname | grep -qi Darwin" "run 'echo Running on OS X'"
# Linux
if-shell "uname | grep -qi Linux" "run 'echo Running on Linux'"
# Cygwin
if-shell "uname | grep -qi Cygwin" "run 'echo Running on Windows'"
{% endhighlight %}

## The Platforms

### Mac OS X
In order to use the clipboard on OS X with tmux you will need to install
[reattach-to-user-namespace](https://github.com/ChrisJohnsen/tmux-MacOSX-pasteboard). (e.g. of installing with [homebrew](http://brew.sh).)
{% highlight bash %}
~/ ❯ brew install reattach-to-user-namespace
{% endhighlight %}

The `reattach-to-user-namespace` command will be used as a wrapper around the `pbcopy`
and `pbpaste` commands.

{% highlight conf %}
# copy
if-shell 'uname | grep -qi Darwin && which reattach-to-user-namespace > /dev/null' 'bind-key -n -t emacs-copy M-w copy-pipe "reattach-to-user-namespace pbcopy"'
# paste
if-shell 'uname | grep -qi Darwin && which reattach-to-user-namespace > /dev/null' 'bind ] run "reattach-to-user-namespace pbpaste | tmux load-buffer - ; tmux paste-buffer"'
{% endhighlight %}


### Linux
On Linux we can use the `xclip` command to handle clipboard synchronization.
{% highlight conf %}
# copy
if-shell 'uname | grep -qi Linux && which xclip > /dev/null' 'bind-key -n -t emacs-copy M-w copy-pipe "xclip -i -sel p -f | xclip -i -sel c "'
# paste:
if-shell 'uname | grep -qi Linux && which xclip > /dev/null' 'bind-key -n C-y run "xclip -o | tmux load-buffer - ; tmux paste-buffer"'
{% endhighlight %}


### Cygwin
The clipboard on [cygwin](http://cygwin.com) is available via `/dev/clipboard`.

{% highlight conf %}
# copy
if-shell 'uname | grep -qi Cygwin' 'bind-key -n -t emacs-copy M-w copy-pipe "cat > /dev/clipboard"'
# paste:
if-shell 'uname | grep -qi Cygwin' 'bind ] run "cat /dev/clipboard | tmux load-buffer - ; tmux paste-buffer"'
{% endhighlight %}

### Putting It All Together

Place the following in `~/.tmux.clipboard.conf`
{% highlight conf %}
# Mac OX X
if-shell 'uname | grep -qi Darwin && which reattach-to-user-namespace > /dev/null' 'bind-key -n -t emacs-copy M-w copy-pipe "reattach-to-user-namespace pbcopy"'
if-shell 'uname | grep -qi Darwin && which reattach-to-user-namespace > /dev/null' 'bind ] run "reattach-to-user-namespace pbpaste | tmux load-buffer - ; tmux paste-buffer"'

# Linux
if-shell 'uname | grep -qi Linux && which xclip > /dev/null' 'bind-key -n -t emacs-copy M-w copy-pipe "xclip -i -sel p -f | xclip -i -sel c "'
if-shell 'uname | grep -qi Linux && which xclip > /dev/null' 'bind-key -n C-y run "xclip -o | tmux load-buffer - ; tmux paste-buffer"'

# Cygwin
if-shell 'uname | grep -qi Cygwin' 'bind-key -n -t emacs-copy M-w copy-pipe "cat > /dev/clipboard"'
if-shell 'uname | grep -qi Cygwin' 'bind ] run "cat /dev/clipboard | tmux load-buffer - ; tmux paste-buffer"'
{% endhighlight %}

Next add a new line to `~/.tmux.conf`
{% highlight conf %}
# System clipboard support
source-file ~/.tmux.clipboard.conf
{% endhighlight %}

## Running tmux Outside of a Window Manager
Chances are that if you are running tmux without a window manager you won't want any of the
custom clipboard configuration. Because we split the clipboard config into a separate file
we can use the `if-shell` tmux command to limit when the `.tmux.clipboard.conf` file is
sourced.

{% highlight conf %}
# Clipboard support when a display is detected
if-shell '[[ -n $DISPLAY ]]' 'source-file ~/.tmux.clipboard.conf'
{% endhighlight %}

# Bonus Round: Emacs in tmux on OS X

Running Emacs inside of tmux on OS X is pretty straightforward; add the following code to your `.emacs` config file to allow copy/paste with the system clipboard.

{% highlight cl %}
(when (eq system-type 'darwin)
  ;; terminal clipboard while inside tmux
  (unless (display-graphic-p)
    (when (and (> (length (getenv "TMUX")) 0) (executable-find "reattach-to-user-namespace"))

    (defun paste-from-osx ()
      (shell-command-to-string "reattach-to-user-namespace pbpaste") )

    (defun cut-to-osx (text &optional push)
      (let ((process-connection-type nil))
        (let ((proc (start-process "pbcopy" "*Messages*" "reattach-to-user-namespace" "pbcopy") ))
		  (process-send-string proc text)
		  (process-send-eof proc))))

      (setq interprogram-cut-function 'cut-to-osx)
      (setq interprogram-paste-function 'paste-from-osx)
    )
  )
)
{% endhighlight %}
