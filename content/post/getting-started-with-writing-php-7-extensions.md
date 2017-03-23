+++
date = "2016-08-05T20:35:54Z"
title = "Getting Started with Writing PHP 7 Extensions"

+++

This blog post is the first in a series of PHP 7 extension building.

You will be joining me on my journey of discovering how extensions are built. I have never written C code before, or looked deeply into PHP internals. I will make mistakes, and I will try my best to correct them as we go along, but if you notice anything, please email me and i'll fix it as soon as possible.

---

# Contents

1. **Getting Started**
2. Hello World
3. Explaining Zend Module Entry

---

## Install Vagrant

The first thing we have to do is install PHP 7 as a development version. The best way to do this is to use a [vagrant](https://www.vagrantup.com/) box  built by [Rasmus Lerdorf](https://en.wikipedia.org/wiki/Rasmus_Lerdorf) - [rasmus/php7dev](https://atlas.hashicorp.com/rasmus/boxes/php7dev).

I will be creating a folder in `~/php-extensions/` for each of the extensions created.

---

## Install Vagrant Box

```bash
$ vagrant init rasmus/php7dev; vagrant up --provider virtualbox
```

Once that has completed, you will want to `vagrant ssh` into the box.

The rest of the series of blog posts will assume we are within the vagrant box.

---

## Check everything is ready

Everything should be ready, but to check type these commands in and you should see something like the below:

```bash
$ which phpize
# /usr/local/bin/phpize
$ php --version
# PHP 7.0.11-dev (cli) (built: Aug  3 2016 23:41:41) ( NTS DEBUG )
```

If you see it all as above then we are ready to continue.

---

## IDE / Text Editor

For PHP extensions you do not need anything special in terms of an IDE, you will be absolutely fine using Sublime Text, Atom, etc. I will be using [CLion](https://www.jetbrains.com/clion/) by [JetBrains](https://www.jetbrains.com).
