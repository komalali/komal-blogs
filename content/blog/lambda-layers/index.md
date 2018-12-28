---
title: Using Lambda Layers to run headless chrome.
date: '2018-12-27T22:13:32.169Z'
---

A few weeks ago I started looking into creating a PDF generation microservice that could be used by various report-building apps that I'm working on. This task appeared to be an ideal candidate for a Lambda function, and thus began my foray into the rabbit-hole that is AWS documentation. It turns out, there's not a lot of good documentation for what I wanted to do. Plus, the Serverless/AWS landscape evolves so fast that what I read from 6 months ago is practically irrelevant at this point. I'm using this post to highlight what I learned and explain why I made some of the decisions I did so that perhaps some other poor soul might not suffer.

## Goal

The specific goal of this service is to create PDFs. The input can be either a URL or a string of HTML, and the output is the PDF file.

The basic idea is, you programatically open a headless browser window and either navigate to a page (if the input is a URL) or set the page content to the input (if the input is HTML). Then, you execute a simple 'print to PDF' with your desired print settings and voila, you have yourself a PDF generator.

Therefore, the more generalized goal of this exercise and the part that I ended up spending most of my time on is *how do I run a chrome instance in a Lambda function and still keep within Lambda's size limitations?*

## Prior knowledge

I started out knowing almost nothing. Basically two things:

   1. I knew that a Lambda function has limits on how big the size of the package that you deploy (50mb compressed) and how long it can run for (15 minutes).
   2. I knew that I had to use a headless browser to create the PDF. [Puppeteer](https://github.com/GoogleChrome/puppeteer) (a Node library that provides a high-level API to run headless chrome) seemed like the best option, as it has a very convenient `Page.toPdf()` function.
