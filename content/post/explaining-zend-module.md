+++
date = "2016-09-05T22:07:42Z"
title = "Explaining Zend Module Entry"

+++

# Contents

1. Getting Started
2. Hello World
3. **Explaining Zend Module Entry**

---

In my [last blog post](https://zando.io/writing-a-hello-world-php-7-extension/) I skirted past what `zend_module_entry` does, and just put the code there. I'm going to attempt to explain it with its many `NULL` values.

This is what I put in the Hello World extension:

```c
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
```

This structure comes from [here](https://github.com/php/php-src/blob/1c295d4a9ac78fcc2f77d6695987598bb7abcb83/Zend/zend_modules.h#L73-L102):

```c
struct _zend_module_entry {
    unsigned short size;
    unsigned int zend_api;
    unsigned char zend_debug;
    unsigned char zts;
    const struct _zend_ini_entry *ini_entry;
    const struct _zend_module_dep *deps;
    const char *name;
    const struct _zend_function_entry *functions;
    int (*module_startup_func)(INIT_FUNC_ARGS);
    int (*module_shutdown_func)(SHUTDOWN_FUNC_ARGS);
    int (*request_startup_func)(INIT_FUNC_ARGS);
    int (*request_shutdown_func)(SHUTDOWN_FUNC_ARGS);
    void (*info_func)(ZEND_MODULE_INFO_FUNC_ARGS);
    const char *version;
    size_t globals_size;
#ifdef ZTS
    ts_rsrc_id* globals_id_ptr;
#else
    void* globals_ptr;
#endif
    void (*globals_ctor)(void *global);
    void (*globals_dtor)(void *global);
    int (*post_deactivate_func)(void);
    int module_started;
    unsigned char type;
    void *handle;
    int module_number;
    const char *build_id;
};
```

## STANDARD\_MODULE\_HEADER
This is a macro that fills in a few of the fields for us.

There are a few of these macros defined, these are:

 - [STANDARD\_MODULE\_HEADER\_EX](https://github.com/php/php-src/blob/1c295d4a9ac78fcc2f77d6695987598bb7abcb83/Zend/zend_modules.h#L43)
  - [STANDARD\_MODULE\_HEADER](https://github.com/php/php-src/blob/1c295d4a9ac78fcc2f77d6695987598bb7abcb83/Zend/zend_modules.h#L44)

These macros fill the fields size, zend_api, zend_debug, zts and, ini\_entry and deps for the non \_EX header

STANDARD\_MODULE\_HEADER is used when you don't have any dependencies, if you have dependencies then use `STANDARD_MODULE_HEADER_EX` instead, and it will leave out a `deps` field that you can fill in.

## PHP\_HELLO\_EXTNAME
This is the name of our extension. We registered this as a constant in `hello.h`, you could simply put a string in here. This field is **required**.

## hello\_functions

This is a reference to the structure of zend\_function\_entry that we put in

```c
zend_function_entry hello_functions[] = {  
            PHP_FE(hello_world, NULL)
        PHP_FE_END
};
```

`zend_function_entry` is just a structure containing one or more [function definitions](https://github.com/php/php-src/blob/d8507d4f582997ae3efe074f54900af2936835e6/Zend/zend_API.h#L76), and ending with a `PHP_FE_END` [source](https://github.com/php/php-src/blob/d8507d4f582997ae3efe074f54900af2936835e6/Zend/zend_API.h#L98).

## The next 5
Are ones you won't touch often, unless you know what you're doing.

They define, in order,

 - Module startup function
  - Module shutdown function
  - Request startup function
  - Request shutdown function
  - Info Function

## Version Number
Next up is the version of the extension. This should be provided as a string, though we are using a constant that we defined in `hello.h`. There is also a constant `NO_VERSION_YET`, which just defines it as `null`.

## STANDARD\_MODULE\_PROPERTIES
The rest of the properties are filled in for you by using the [STANDARD\_MODULE\_PROPERTIES](https://github.com/php/php-src/blob/1c295d4a9ac78fcc2f77d6695987598bb7abcb83/Zend/zend_modules.h#L61). There should be no need for you to fill in these, unless you need to fill in some extension globals, in which case use `STANDARD_MODULE_PROPERTIES\_EX`.
