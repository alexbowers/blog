+++
title = "Laravel 5.3  - Multiple Migration Paths"
date = "2016-05-25T21:21:08Z"

+++

In Laravel 5.3+ there is a new feature that allows for multiple migration paths.

## Why should I care?

You may not think its that big a deal, but it will help many packages significantly.

A required step previously was to have a publication step, or a copy paste step for the migrations from `vendor/package/database/migrations` to `database/migrations`.
This new feature allows the migrations to stay where they are, and not clutter up your migrations folder.

This is useful because when a package gets a new version, you don't want to possibly have to copy their migrations over again, you want to be able to just run `php artisan migrate` and have it work.

In this new system it will.

## Okay that sounds great. How do we use it?

As soon as you are on Laravel 5.3, it will be enabled for you by default. All you have to do is in a Service Provider (for example, a MigrationsServiceProvider) `boot` method, is call `$this->loadMigrationsFrom('path/to/migrations/folder')`.

Any subsequent calls to `php artisan migrate` will just work.

## Erm, I don't know. Do I have to do this?

If you do not wish to use this feature, nothing has changed. You can still copy migrations from the package to your database migrations folder if you wish to.
And you do not have to add your migration path in a service provider, that's done for you.

## What should use this?

This should be used by any system that wants to use it.

If any package you use requires you to pull in their migration files from a specific location, then this is a prime candidate for this to be used.

If your aiming for a modular design.
This is a must to keep the core separate from each module.
Now, your modules are just composer directories living in their own space, and there is no bleeding across the two.
