+++
date = "2016-08-05T21:26:29Z"
title = "Writing a Hello World PHP 7 Extension"

+++

# Contents

1. Getting Started
2. **Hello World**
3. Explaining Zend Module Entry

---

## Getting Setup

In the folder `/vagrant` you will want to create a new folder with the name of your extension. I will be calling my extension `hello` and so my folder will be `hello-ext`. This is just convention, and the folder name doesn't actually matter.

```bash
$ mkdir /vagrant/hello-ext
```

A very basic PHP extension requires 3 files:

```
config.m4
src/hello.c
src/hello.h
```

The `config.m4` extension tells PHP some basic information about your extension. The `hello.c` and `hello.h` files are the actual extension itself.

---

## Actually writing the extension

### Test case first

To check whether this actually works, this is a basic PHP file we using to check:

```php
<?php
var_dump(extension_loaded('hello'));

var_dump(hello_world());
?>
```

At every step of the way through this process, you can re-run this file using `php -f /path/to/test/file.php`. If you run this now, you'll see:

```bash
$ php -f test.php
# bool(false)

# Fatal error: Uncaught Error: Call to undefined function hello_world() in /vagrant/test.php:5
# Stack trace:
# #0 {main}
#    thrown in /vagrant/test.php on line 5
```

Note that I hold this file **outside** of the `ext` folder. This file isn't part of our extension. We will deal with proper testing at a later date.

### config.m4

The first thing we should tell PHP about is the argument to enable the `hello` extension.

The syntax for the `config.m4` file is based on the `GNU autoconf` [syntax](https://www.gnu.org/software/autoconf/manual/autoconf.html). Though, you don't need to know it really. There will be very few lines involved, and i'll do my best to explain them.

Comments can be left in the `config.m4` file using `dnl`


```autoconf
dnl Tell PHP about the argument to enable the hello extension

PHP_ARG_ENABLE(hello, Whether to enable the hello world extension, [ --enable-hello   Enable hello world])

if test "$PHP_HELLO" != "no"; then
    PHP_NEW_EXTENSION(hello, src/hello.c, $ext_shared)
fi
```

`dnl` stands for `Discard to Next Line` <sup>[1](http://stackoverflow.com/questions/3371239/autoconf-dnl-vs#comment48017868_3377708)</sup>, including the new line at the end of the line.

---

Breaking `PHP_ARG_ENABLE(hello, Whether to enable the hello world extension, [ --enable-hello   Enable hello world])` down:

`PHP_ARG_ENABLE` - Provides `configure` with the options and help text seen when running `./configure --help`.

There is also a `PHP_ARG_WITH`, and either `ENABLE` or `WITH` should be provided. The difference there is how `configure` gets ran, either `--with-*` or `--enable-*`.


> **--enable-SOMETHING** is used for extensions with zero dependencies

> **--with-SOMETHING** is used for extensions with dependencies (e.g. it's --enable-pdo, but --with-pdo-mysql as the latter may depend on libmysqlclient)

*Thanks to /u/dshafik for clarification on this*

> Every extension should provide at least one or the other with the extension name, so that users can choose whether or not to build the extension into PHP.

---

> By convention, `PHP_ARG_WITH()` is used for an option which takes a parameter, such as the location of a library or program required by an extension, while `PHP_ARG_ENABLE()` is used for an option which represents a simple flag.

[source](http://php.net/manual/en/internals2.buildsys.configunix.php)

`hello` is just the name of our extension.

`Whether to enable the hello world extension` Not entirely sure what this is for to be honest.

`[ --enable-hello   Enable hello world]` this determines the output for `configure --help`.

It will end up looking something like this:

![Image](http://i.imgur.com/ygZBnfD.png)

being what was inside the square brackets.

---


Then next bit is literally just a basic `if` check to determine whether or not the the extension is enabled. By default, it will be for us.

Now we need to register the extension.

`PHP_NEW_EXTENSION(hello, src/hello.c, $ext_shared)`



> The first parameter is the name of the extension.

> The second parameter is the list of all source files which are part of the extension.

> The third parameter should always be $ext_shared, a value which was determined by configure.

Taken, and adapted from the woefully outdated [php guide](http://php.net/manual/en/internals2.buildsys.configunix.php).

There are more parameters, but so far we don't need them, so we will leave them for a different day.

Now to the actual extension itself.

---

### src/hello.h

This file is quite basic:

```c
#define PHP_HELLO_EXTNAME "hello"
#define PHP_HELLO_VERSION "0.0.1"

PHP_FUNCTION(hello_world);
```

We are defining two directives, what this allows us to do is use `PHP_HELLO_EXTNAME` in place of the actual string `hello` throughout.

This is common practice, so whilst not necessary strictly in this use case, it is a good habit to have.

The first interesting line is `PHP_FUNCTION(hello_world);`

This is where we define the header for our function.

The body of the function will go within our `hello.c` file.

---

### src/hello.c

This is where things get a little more complicated now.

So i'll give you the full file, and then break it down a little:

```c
#include <php.h>
#include "hello.h"

zend_function_entry hello_functions[] = {
            PHP_FE(hello_world, NULL)
        PHP_FE_END
};

zend_module_entry hello_module_entry = {
            STANDARD_MODULE_HEADER,
        PHP_HELLO_EXTNAME,
        hello_functions,
        NULL,
        NULL,
        NULL,
        NULL,
        NULL,
        PHP_HELLO_VERSION,
        STANDARD_MODULE_PROPERTIES,
};

ZEND_GET_MODULE(hello);

PHP_FUNCTION(hello_world) {
            php_printf("Hello world\
");
};
```

Pasting this in to your file, and then going to the terminal and running the following commands will allow this to work.

```bash
$ cd /vagrant/hello-ext
$ phpize
$ ./configure
$ make
$ sudo make install
```

The next thing to do is add the extension into your `php.ini` file.

This will look like so:

```ini
extension=hello.so
```

Now, go back to your test file and run it again.

You should see something like this:

```bash
$ php -f test.php
bool(true)
Hello world
NULL
```

> Note: Fixed typo with response above, thanks to /u/phisch90

Success.

The reason for the hanging `NULL` is because we are using the `printf` function, rather than returning the value.
