---
author: Paul Elias Sojan
title: Before your next Rails upgrade
date: 2024-03-17
tags:
  - ruby on rails

readingSpeed: 20
readingSpeedMin: 50
readingSpeedMax: 100
---

If you are planning to upgrade your Rails app, which has a lot of active development. Let's say your current rails version of your app is `6.1.7` and you are planning to upgrade it to `7.1.1`. The default upgrade strategy that most people use when updating their Rails apps is called the `Long Running Branch` strategy, which means creating a separate branch from the `main`, which provides a dedicated environment for developers to work on the upgrade. This isolation allows developers to experiment, iterate, and collaborate on complex changes without disrupting the mainline development workflow.

However, challenges arise when other developers merge their changes into the `main` branch. Because you’ve been working on a separate branch for so long, but other people are concurrently making other changes. When they merge those changes, it ends up in conflict with your upgrade.

This is where the `dual boot` strategy comes into play. The idea is to support both versions of Rails on your app at the same time by maintaining separate `Gemfile.lock` files for each version. Keeping the application runnable on the current Rails version guarantees that you will be able to run the application on it if something goes wrong.

With this setup, there is no need to maintain a separate branch until the upgrade is fully complete, we can merge those works frequently to `main`. This eliminates the headache of Git conflicts and the need for back-merging other branches. Also, you can run separate CI jobs for the current and next versions of Rails. Once the incremental upgrade is done, you can deploy the next Rails version to production.

### How to dual boot your app

To accomplish dual boot setup, we need to create two `Gemfile.lock` files: one corresponding to the current version `Gemfile.lock` and another for the target version `Gemfile_next.lock`. To load dependencies from environment-specific lock files, we need to monkey patch the `Bundler`. Environment variables are used to indicate which version of Rails the application should load.

We will be using a gem called [bootboot](https://github.com/Shopify/bootboot) to do the monkey patch for us.

To implement the monkey patch using `bootboot` follow the below steps:

Step1: Add `bootboot` to your Gemfile

```
plugin 'bootboot', '~> 0.2.1'
```

Step2: Install the gem

```
bundle install && bundle bootboot
```

This command also creates `Gemfile_next.lock`.

Now, if you go to the `Gemfile` the below line gets added:

```
if ENV['DEPENDENCIES_NEXT']
  enable_dual_booting if Plugin.installed?('bootboot')

  # Add any gem you want here, they will be loaded only when running
  # bundler command prefixed with `DEPENDENCIES_NEXT=1`.
end
```

`DEPENDENCIES_NEXT` is the ENV variable for determining the target version. If you want to boot using the dependencies from the `Gemfile_next.lock`, run any bundler command prefixed with the `DEPENDENCIES_NEXT=1` ENV variable.

Now add the target rails version to the if condition and the current version in the else condition. For example, let's say the target version is `7.1.1` and the current version is `6.1.7`.

```
if ENV['DEPENDENCIES_NEXT']
  enable_dual_booting if Plugin.installed?('bootboot')

  gem "rails", "7.1.1"

else
  gem "rails", "6.1.7"
end
```

To install the target version of rails run:

```
DEPENDENCIES_NEXT=1 bundle install
```

The new version gets locked into the `Gemfile_next.lock` file.

If you find an error or deprecation warning in the existing code while running your application in the target version, you can conditionally wrap those areas like this:

```
def some_method
  if ENV['DEPENDENCIES_NEXT']
    # new behavior
  else
    # old, deprecated behavior
  end
end
```

Once the current incremental upgrade is complete you can remove the conditions, preserving only the code in the if condition.

Now, let's check how the Rails server responds when started with and without `DEPENDENCIES_NEXT` ENV.

With `DEPENDENCIES_NEXT`:

![with-dependency](https://ik.imagekit.io/eapzn8piu/with-dependecy.png)

Without `DEPENDENCIES_NEXT`:

![without-dependency](https://ik.imagekit.io/eapzn8piu/without-dependecy.png)

Now that you can seamlessly switch between Rails versions on the fly 🎉 🎉.
