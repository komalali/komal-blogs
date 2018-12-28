---
title: Using Lambda Layers to run headless chrome.
date: '2018-12-27T22:13:32.169Z'
---

A few weeks ago I started looking into creating a PDF generation microservice that could be used by various report-building apps that I'm working on. This task appeared to be an ideal candidate for a Lambda function, and thus began my foray into the rabbit-hole that is AWS documentation. It turns out, there's not a lot of good documentation for what I wanted to do. Plus, the Serverless/AWS landscape evolves so fast that what I read from 6 months ago is practically irrelevant at this point. I'm using this post to highlight what I learned and explain why I made some of the decisions I did so that perhaps some other poor soul might not suffer. I'm going to start off by just talking through the problem and how I solved it, and will end with all the code necessary to make this work.

## Goal

The specific goal of this service is to create PDFs. The input can be either a URL or a string of HTML, and the output is the PDF file.

The basic idea is, you programatically open a headless browser window and either navigate to a page (if the input is a URL) or set the page content to the input (if the input is HTML). Then, you execute a simple 'print to PDF' with your desired print settings and voila, you have yourself a PDF generator.

But really, printing to PDF is only one thing you can do with a headless chrome instance. You can also use it to take screenshots, or run functional tests, or probably a whole bunch of other things I haven't had to think of yet. Therefore, the more generalized goal of this exercise and the part that I ended up spending most of my time on is *how do I run a chrome instance in a Lambda function and still keep within Lambda's size limitations?*

## Prior knowledge

I started out knowing almost nothing. Basically two things:

1. I knew that a Lambda function has limits on how big the size of the package that you deploy (50mb compressed) and how long it can run for (15 minutes).
2. I knew that I had to use a headless browser to create the PDF. [Puppeteer](https://github.com/GoogleChrome/puppeteer) (a Node library that provides a high-level API to run headless chrome) seemed like the best option, as it has a very convenient `Page.toPdf()` function.

## The problem

So now we get into the fun stuff. In the perfect world, this would be just as simple as adding the puppeteer package to our project and using it in our function. But! when you install puppeteer into your project, it **also downloads a recent version of Chromium(~170MB Mac, ~282MB Linux, ~280MB Win) that is guaranteed to work with the API**!!

This is great for our local development, but way too big to fit in the compressed package that we are deploying to AWS Lambda. Plus, even if it were small enough, Puppeteer installs the version that makes sense for your local environment (in my case a mac) and that's not the same OS as what is running in my Lambda (AWS Linux I think?).

**So now, we need:**

1. A chrome binary that runs on AWS Lambda.
2. Some way to not include the local chromium that puppeteer downloads in the package we deploy to AWS Lambda.
3. Be able to use the correct chromium depending on the environment the function is running in (so we can test our functions locally without having to redeploy after every change).

## The solution++

* Okay so lucky us! There is already a chrome binary for AWS as a node package called [chrome-aws-lambda](https://www.npmjs.com/package/chrome-aws-lambda). And it is just 34mb! Sweet! That makes it much more likely that we can fit all the required code within the 50mb limit.
* And cool, there is a serverless option that allows us to exclude certain files and folders from our package, so we can use that to exclude the local chrome version.
* And fine, I guess we can just check which OS our function is running on, and choose which chrome to use accordingly - that seems straightforward enough.

But, to be honest, 34mb is still quite a chunk. Plus, it's definitely big enough that Lambda won't let you edit your function through the AWS console. Really, the chrome binary doesn't have anything to do with the function itself, and so it's kinda just dead weight that we have to wait to upload every time we deploy our functions. What would be really awesome is if there was a way to have the chrome binary exist as a separate piece of code that my function could just connect to when needed.

And **as of November 29th, 2018 all of this is possible via [Lambda Layers](https://docs.aws.amazon.com/lambda/latest/dg/configuration-layers.html).** Awesome. So now we can extract the `chrome-aws-lambda` dependency into its own layer, and this will speed up our time to deploy our functions, make them easier to debug in the AWS console, and expose the layer for use by other Lambdas that might also need to run chrome for one of the other use cases I discussed earlier.

Alright, enough talking let's see the code.

## The code