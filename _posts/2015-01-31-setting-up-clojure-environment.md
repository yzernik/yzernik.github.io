---
layout: post
title:  "Setting up Emacs for Clojure"
categories: jekyll update
---

Now that I am learning Clojure, I need to have a properly configured environment for Clojure development.

With Clojure, Emacs is the de-facto standard editor.

I've been using Emacs for a long time, but in that time I've only scratched the surface of the powerful features that are available.

My usual workflow consists of basic commands like `open-file` (C-x C-f), `save-buffer` (C-x C-s), `save-buffers-kill-emacs` (C-x C-c), and copy-paste. I also like to use the `isearch-forward` command (C-s) and the `query-replace` command (M-%). I've recently started using `goto-line` (M-g g).

I still have a long way to go to become a real Emacs nerd.

## Clojure and Emacs

My Emacs skills have stagnated in recent years. When I program in Scala, I use the Scala IDE for Eclipse. When I program in Java, I use IntelliJ IDEA at work, and Eclipse for personal projects. I mostly only use Emacs for Python and C programming.

There is an [excellent guide][brave-and-true-emacs] to using Clojure with Emacs in one of the chapters of *Clojure for the Brave and True*. This guide is based on an existing [emacs configuration][brave-and-true-emacs-conf] created by the author.

Since I already have my own [Emacs configuration][yzernik-emacs-d], and my goal is to learn both Emacs and Clojure, I've decided to update my own repository to keep it compatible with the requirements of the book.

## Managing Emacs packages

My `.emacs.d` directory is loosely based on [another one][purcell-emacs-d] that I found on Github. I stripped out almost everything except for the structure of the project, and the `init.el` file. I also kept the `init-elpa.el` file (renamed to `init-melpa.el`, since I'm using MELPA, and I don't know the difference).

MELPA (*Milkypostman's ELPA or Milkypostman's Experimental Lisp Package Archive*) seems to be the most widely used Emacs package repository archive. So far, I have found every package I need [there][melpa].

The `init-melpa.el` file is key because it bootstraps the package installation process by accessing the online package archive. It also contains one other important piece, which is a function called `require-package`. This will check if a given package is already installed, if its contents need to be refreshed, and if the `package-install` command should be run. This turns out to be much easier and more portable than manually running `package-install` on every new machine that uses the config.


```elisp
(defun require-package (package &optional min-version no-refresh)
  "Install given PACKAGE, optionally requiring MIN-VERSION.
If NO-REFRESH is non-nil, the available package lists will not be
re-downloaded in order to locate PACKAGE."
  (if (package-installed-p package min-version)
      t
    (if (or (assoc package package-archive-contents) no-refresh)
        (package-install package)
      (progn
        (package-refresh-contents)
        (require-package package min-version t)))))
```


With this infrastructure in place, I was able to create the configuration for the *major modes* that I will most commonly use. I have `tuareg` mode for OCaml, `ensime` for Scala, `markdown-mode` for editing Markdown, and of course, `clojure-mode`.



[brave-and-true-emacs]:       http://www.braveclojure.com/basic-emacs/
[brave-and-true-emacs-conf]:  https://github.com/flyingmachine/emacs-for-clojure
[yzernik-emacs-d]:            https://github.com/yzernik/.emacs.d
[purcell-emacs-d]:            https://github.com/purcell/emacs.d
[melpa]:                      http://melpa.org
