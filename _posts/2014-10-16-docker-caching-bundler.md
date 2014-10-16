---
layout: post
title: Caching bundler when building a ruby/rails app's image with docker
---

**Docker caching mechanism**

Docker has an automatic caching mechanism to greatly speed things up after the first build of a Dockerfile. Each step (each docker instruction of the file) is cached separately. If you change a docker instruction, Docker will pull the results out of the cache of its previous step. It can be really handy, especially if your building ruby from sources or have a lot of huge dependencies like Nokogiri and co.

**Bundler caveat**

With a Rails app, an obvious caveat appears right after a few rebuild of an image: 

You have to sit and wait for Bundler to finish installing every dependencies, even if the Gemfile/Gemfile.lock haven't changed at all.

It's not a viable solution to keep it like this, Bundler can take a lot of time, and example, if
you run your test suite inside a container, and your are developing a new test, you will need to rebuild the image very often, waiting nervously in your desk that Bundler finished its job.

**How can we cache the bundle install step?**

It's pretty simple to use docker's automatic caching mechanism to cache the bundle install.

We will start with this Dockerfile based on the official language stack repository who add the whole Rails app inside the image:

```
FROM ruby:2.1.2

# Install bundler.
RUN gem install bundler
 
# Now copy the app into the image.
COPY . /rails
 
# Install ruby gems.
RUN cd /rails && bundle install

# Set the final working dir to the Rails app's location.
WORKDIR /rails
```

We just need to add Gemfile/Gemfile.lock inside the image before bundle install:

```
FROM ruby:2.1.2

# Install bundler.
RUN gem install bundler

# Install ruby gems.
RUN cd /rails && bundle install

# Copy the Gemfile and Gemfile.lock into the image.
COPY Gemfile /rails/Gemfile
COPY Gemfile.lock /rails/Gemfile.lock

# Install ruby gems.
RUN cd $APP && bundle install

# Everything up to here was cached. This includes
# the bundle install, unless the Gemfiles changed.

# Now copy the app into the image.
COPY . /rails

# Set the final working dir to the Rails app's location.
WORKDIR /rails
```

Now, the rebuild of this image won't run Bundler again if you modify a file in your app, unless
your are changing something in either the Gemfile or Gemfile.lock (or the Dockerfile itself).

Happy hacking!