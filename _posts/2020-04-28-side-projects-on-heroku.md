---
layout: post
title:  "Bootstrapping Side Projects Without Breaking the Bank: Heroku Edition"
author: jeff
categories: [ heroku, side-projects, bootstrap ]
tags: []
image: assets/images/heroku-logotype-horizontal-white.jpg
description: "When it comes to side projects, I am constantly seeking to find the perfect balance between cost and efficiency. Here is what I found while working with Heroku."
featured: true
hidden: false
---

Sometimes the most efficient way to launch a side project is also the most expensive. When I say efficiency here, I am referring to the speed at which you can launch a quality product. When it comes to a side project, you want to be able to build something and actually launch it. The problem is that this can often become an expensive endeavour between the cost of hosting, domains, SSL certificates, CI, etc. Luckily there are ways to accomplish each of these things with low or no cost.

In this post, I am going to talk about my recent experience with launching gif2zoom.com on Heroku. Gif2Zoom is a product that allows users to select gifs (from Giphy's free API) and converts them to Mp4 files. These Mp4 files can be used as virtual backgrounds in Zoom. This project contained both a frontend and a backend that needed to be deployed.

## Why Heroku?

I chose Heroku for hosting mainly because of all the things I have heard about it. I have heard that it is easy to use and quick to setup. These sound like great things for a side project! So I decided to dive in and give it a shot, even though I am more familiar with AWS.

## Heroku Pricing

Looking at Heroku's pricing, I realized that I would definitely want to try to stick to their free tier if at all possible. Their free tier gives you access to most of the features that you need for a side project. It includes all of the basics that you need along with support for custom domains and application logging, though it does have a few deficiencies:

- No support for SSL on custom domains
- Sleeps after 30 minutes of inactivity
- Pulls from an allotment of "free hours" that you have remaining in a given month (you are given about enough hours to run a single service all month long)
- No application metrics

You can overcome all of these deficiencies by upgrading to Heroku's "Hobby" Dyno (Dyno is their word for a node or instance). However, I don't want to pay $7 a month for a side-project like this one. This will definitely vary from project to project, but in general I want to keep the hosting costs as low as possible. Further, $7 a month seems a little steep considering that the instance they give you only has 512MB of RAM. In any case, I decided that these constraints were acceptable for this project and moved on to getting started.

## Gif2Zoom.com Deployment

Due to the restriction on the "number of free hours" that Heroku imposes, I chose to consolidate my frontend and backend into the same deployment. This way I could host the entire app from one Dyno. If I were to host from two Dynos, I would be twice as likely to run out of free hours in a given month. The downside of this decision is that I now will have much more traffic coming into this single Dyno. I will have to update this post to tell you how my single Dyno holds up once I have had it released for a little while.

There are a few options for deployment to Heroku. I chose to follow the approach of using a plugin to my build tool. I am not going to go into too much depth on this point since Heroku has great documentation on how to deploy different application stacks.

I will mention that I did run into a problem after I got my code deployed. Looking at my application logs, I could see that my application was booting correctly and binding onto the correct port. However, I was unable to reach my application at the URL Heroku provided me after deployment. When I tried to reach my application, Heroku was logging out an error with code H10. This means that they are unable to connect to my application. I almost gave up and moved to AWS at this point because nothing I tried seemed to remedy this situation.

<div style="width:23em;margin-left: auto;margin-right: auto;margin-top:20px;margin-bottom:20px;">
<blockquote class="twitter-tweet"><p lang="en" dir="ltr">Heroku has an amazing CLI. The fact that you run 1-3 commands and have an app running in the cloud is way cool.<br><br>That being said, my app keeps crashing on boot without anything in logs..</p>&mdash; Jeff Lewis (@lewisjkl) <a href="https://twitter.com/lewisjkl/status/1254559408454950912?ref_src=twsrc%5Etfw">April 26, 2020</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>
</div>

In the end, I realized that I was binding to the wrong network interface. I was binding to localhost (127.0.0.1) instead of 0.0.0.0. For those not familiar, 0.0.0.0 tells the OS to listen on every network interface on the port you have specified. Once I changed my application to bind on 0.0.0.0 everything worked great.

tl;dr If you are having connection issues even though you are binding on the correct port, check which network interface you are binding to.

## Custom Domain

When you first deploy your application to Heroku, they will provide you with a url of the form `https://<your-app>.herokuapp.com/` to reach your application. Although you could release your application using this URL, I wanted to have a custom URL. I chose to use GoDaddy to get a custom URL since I manage several other domains through them.

The domain gif2zoom.com cost $12.17. I opted into adding privacy on top of my domain so it doesn't need to get registered to my home address directly. This cost another $9.99 which brought my total cost plus tax to $22.88. This cost will end up being yearly if I choose to renew the domain, but it is less than $2 a month so I was willing to spend it for a good domain name.

## Configuring DNS for Custom Domain

I learned the hard way, that Heroku and GoDaddy do not play nicely with each other. If you are using GoDaddy with Heroku, you will not be able to point your domain directly to your site except by using a subdomain like "www." In other words, "www.gif2zoom.com" would have worked, but "gif2zoom.com" (without the "www") would have failed to point at my site.

There are two ways around this issue. One is to buy your domain from a provider on Heroku's [list](https://devcenter.heroku.com/articles/custom-domains#add-a-custom-root-domain). The way that I chose to go, however, was to use a Heroku add-on called PointDNS. PointDNS basically allows you to work with Heroku as you would expect it to work out of the box. Lucky for me, you are able to add a single domain using PointDNS for free. After that you need to pay $15 dollars a month.

Setting up PointDNS wasn't too bad. They provide you with a few name servers that look something like `dns12.pointhq.com.`. I put those into GoDaddy under my domain.

<div style="width:23em;margin-left: auto;margin-right: auto;margin-top:20px;margin-bottom:20px;">
  <img src="{{ site.baseurl }}/assets/images/nameservers.png"/>
</div>

After adding these I was finally able to reach my site at my domain and everything worked!

## Heroku - Recap

All in all, setting up my application on Heroku was really smooth and I was able to do it without any costs (other than domain registration). Here is a recap of Heroku along with some other thoughts about it:

### The Good

- The CLI. Heroku's CLI is awesome. It allows you to deploy applications, restart your applications, and view application logs.
- Documentation is concise and well done.
- Quick to set up. It is hard to find a way to get an application up and running more quickly.

### The Less Good

- No native support for using DNS from certain providers (such as GoDaddy). You can use something like PointDNS, but that will be expensive for anything more than a single app.
- No support for SSL with free tier.
- Expensive once you move off of the free tier (getting an instance with 1GB of RAM will cost $50 a month).

## Conclusion

In the end, I won't be using Heroku for my next side project. I think it works reasonably well for a single side project (if it can run on the free tier), but once you need to host more than one project it will get really expensive. You may need to pay for things like PointDNS and bigger instances in Heroku which are both expensive.

For my next project, I am going to use AWS to host. I will write another post talking about my findings and comparing the experience to Heroku. I am hoping that I will be able to setup AWS in such a way that it will be just as easy to deploy to as Heroku after I get everything setup the first time. If I can achieve that along with a lower end cost, then I will consider it a success and deploy all my future side projects that way.

Follow me on Twitter to keep up with my latest projects, posts, and findings.
