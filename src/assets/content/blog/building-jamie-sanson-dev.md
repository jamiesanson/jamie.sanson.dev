---
title: Building jamie.sanson.dev
author: Jamie Sanson
date: 2019-12-24T06:15:00.000+00:00
hero_image: "/src/assets/content/images/dallas-at-night-2332288.jpg"

---
Everyone big in tech has their own site, right? Who _doesn't_ maintain their own blog post site? At some point in time, it sounds like a good idea to spin up something of your own. I toyed with the idea of doing so for quite some time, but I'm not a web developer, so it always landed in the hard basket.

However, things have changed recently with the introduction of more and more free tools to take the pain away, and integrate directly with Git in the process. Let's have a look at what it takes, at this day in age, to spin up a site like this. This post is written with the assumption that you use Github for change management.

# Ingredients of a blog site

There's a few key components that go in to building and maintaining your own, personalised blog site:

* A custom domain, for making access to your site branded
* A hosting platform, for serving your site
* A CMS, for managing and editing content

These ingredients, when mixed together, leaves you with a site which is easy to promote and maintain. These components are getting cheaper and easier to use, and in doing so are becoming more and more integrated.

# Getting the ingredients together

Gathering the components to make jamie.sanson.dev involved a lot of research into tooling that's new and upcoming, as well as research into which providers for things like domains are reputable. Here's where I found my components, and what they all cost. Spoilers, only one of them had me reaching for my credit card!

## Custom domain: [Namecheap](https://www.namecheap.com/)

There are a plethora of Domain Name Registrars to choose from, some more or less reputable. I didn't look for the best of the best when it came to choosing a registrar to go with, so instead found a well liked one which offers good prices on domains. Namecheap offered sanson.dev at a competitive rate, and has an easy to use GUI for updating things like DNS records, so was my choice. This is the one component which costs money - $14.98 USD a year.

## Hosting platform: [Netlify](https://www.netlify.com/)

Again, with hosting, there's a wealth of options to choose from. Netlify offer their services for _free_ for small, personal sites. They integrate directly with things like Github, to build and deploy static sites with ease when things change in the trunk branch. Netlify makes it easy to build and test your site with each change, and it makes integration with custom domains a breeze.

## Content Management: [Forestry](https://forestry.io/)

Forestry is an incredible new content management system which integrates with git version control systems to allow for preview and editing of content. Their web-browser based content editor allows users to create posts, and edit anything about their site - from authors to application config. When a change is made to non-draft items, that change is committed to source control. That commit triggers the hosting platform to build and deploy the site, completing the circle!

Forestry not only allows you to create and edit attributes of your site, but it also allows you to import entire site templates to get started - easy! jamie.sanson.dev is currently running the [Gridsome Brevifolia](https://github.com/kendallstrautman/brevifolia-gridsome-forestry) theme from @kendallstrautman. Gridsome is a static site generator, built on top of the latest web technology, using things like Vue.js and GraphQL

# Putting it all together

Assembling the site couldn't be easier, using all the technologies above. It can be summarised like this:

1. Grab yourself a domain
2. Get your Github repo set up using a Forestry starter template
3. Create a Netlify project
4. Link your Netlify project to your custom domain by updating a DNS record

And that's it! The only ongoing cost is your custom domain, which renews for less $15 yearly.