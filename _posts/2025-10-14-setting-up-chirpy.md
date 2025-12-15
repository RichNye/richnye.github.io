---
title: Configuring Jekyll and Chirpy in Windows
date: 2025-10-14 11:10:00 +0100
categories: [jekyll]
tags: [blog,jekyll,github]
author: richardnye
description: Configuring Jekyll and Chirpy from scratch in Windows and WSL
---

As I alluded to in my first blog post, I ran into some issues when I first attempted to build the Chirpy starter template to test it locally on my Windows 11 PC. Today's post will be about documenting the steps that I had to take that weren't referenced in the [excellent video from TechnoTim](https://www.youtube.com/watch?v=F8iOU1ci19Q). 

## Configuring WSL
The standard out-of-the-box WSL Ubuntu environment wasn't enough to get Chirpy running locally. I constantly hit errors like the following:
```
Gem::Ext::BuildError: ERROR: Failed to build gem native extension. current directory: /var/lib/gems/3.2.0/gems/json-2.15.1/ext/json/ext/generator /usr/bin/ruby3.2 -I/usr/lib/ruby/vendor_ruby extconf.rb checking for whether -std=c99 is accepted as CFLAGS... *** extconf.rb failed *** Could not create Makefile due to some reason, probably lack of necessary libraries and/or headers. Check the mkmf.log file for more details. You may need configuration options.
```

This was after installing ruby and ruby-dev. It turns out I was also missing a few dependencies, and running this fixed it:

```
apt install build-essential libssl-dev zlib1g-dev libreadline-dev libyaml-dev
```

I also installed ruby-bundler, which is needed to run the bundle commands. Once these packages were installed, I could then run commands such as `bundle exec jekyll s` to test the site locally.

## Customising the template
Once I'd got the site up and running locally, I quickly found myself wanting to make changes. I won't cover all of them, and most (such as removing certain social media icons) were simple enough, but here's the more difficult ones that required a bit of research:

- Removing the Chirpy/Jekyll note in the footer of every page
- Adding my custom image

### Customising the footer
So because you typically don't clone the main Chirpy repo you don't get all site files included - it's typically advised to use the Chirpy starter which includes GitHub Action workflows and other useful bits. This meant I needed to create the _includes directory and download the footer.html file from the main repo. Once I had it, it was simple to remove the elements I didn't want. A simple step with hindsight, but it did take a bit of research when you're completely new to the world of Jekyll and Chirpy.

### Adding my custom image
Again, with hindsight this is so simple to understand, but it did take a bit of learning. Initially, I was adding my png file to where the other site assets were because that would make sense right? Just drop the png into _site/assets and job done?! Wrong. It turns out that directory is rebuilt with every build, and actually gets assets from the root assets directory of the project. So I'd put my png in, build the site, and get confronted with... no image. It was only when I realised the photo was being removed with every build that I realised it needed to go elsewhere.

## Future site adjustments
I'd like to tackle the following:
- Remove or tweak the Categories/Tags sections - I'm not sure they're really what I'm going for here.
- Tweak the About section - rename it to 'About Me' and add content.
- Remove the 'Some rights reserved' message from the footer - I'm not a fan of it. 
- Change the mailto: link - it opens in a new page, and generally seems more complicated than it needs to be?

But we'll see. For now, I can't recommend Chirpy enough. I may move away from it in future, I've seen Hugo getting a lot of attention, but that's a problem for future me.
