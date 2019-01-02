---
title: Puppeteer or chrome-remote-interface?
date: '2019-01-01T00:13:32.169Z'
---

If you're trying to figure out how to run headless chrome programatically via node.js, your two options are Puppeteer or chrome-remote-interface. Here I'll describe each, and talk about specific use cases where you might use one over the other.

### Wait, why?

Hold on a minute. **Why would you wan to run chrome programatically anyway?** Well, it turns out there are lots of reasons! Some examples are:

* Taking screenshots of pages or portions of pages.
* Creating PDFs of pages or HTML.
* Crawling pages for web scraping.
* Running functional (integration) tests.

### Puppeteer

[Puppeteer](https://developers.google.com/web/tools/puppeteer/) is a Node library which provides a high-level API to controll headless Chrome or Chromium. Watch out though, because when you add it to your project (using `npm install puppeteer`), it also downloads a chromium binary that is certain to work with the operating system and the API by default. This binary ranges between 180-280mb depending on the OS, and can significantly increase the size of your project folder. It is recommended for quickly and painlessly setting up chrome-related tasks that don't require a lot of customization, as it hides away the complexity of the chrome DevTools protocol.

**Recommended use** - Common tasks that do not require a lot of customization.

**Example** - Create a PDF of a page.

```javascript{lineNumbers: true}
const puppeteer = require('puppeteer');

(async() => {
    const browser = await puppeteer.launch();
    const page = await browser.newPage();
    await page.goto('https://www.google.com', { waitUntil: 'networkidle2' });
    await page.pdf({ path: 'page.pdf', format: 'A4' });

    await browser.close();
})();
```

### chrome-remote-interface

If you require more fine-grained control that involves using the Devtools protocol directly, [chrome-remote-interface](https://www.npmjs.com/package/chrome-remote-interface) is the tool for you. Keep in mind though, `chrome-remote-interface` doesn't actually launch Chrome for you (or even download a convenient binary for you to launch), so you'll have to handle downloading and launching chrome yourself. You can use [chrome-launcher](https://www.npmjs.com/package/chrome-launcher), a different npm package, to launch a chrome instance. Then, you can talk to the instance of chrome using `chrome-remote-interface`.

**Recommended use** - Custom, specific or fine-grained browser-related tasks that require direct use of the DevTools protocol.

**Example** - Create a PDF of a page. (For comparison, I am using the same example so you can see the difference in code.)
```javascript{lineNumbers: true}
const launcher = require('chrome-launcher');
const CDP = require('chrome-remote-interface');
const fs = require('fs');

(async() => {
    const chrome = await launcher.launch();
    const client = await CDP();
    const { Page } = client;
    await Page.enable();
    await Page.navigate({ url: 'https://www.google.com' });
    const { data } = await Page.printToPdf({
        marginTop: 0,
        marginBottom: 0,
        marginLeft: 0,
        marginRight: 0,
    });
    fs.writeFileSync('page.pdf', Buffer.from(data, 'base64));
    await client.close();
})();
```

### Sources

There's a lot more to learn about the topic and the [google developer docs](https://developers.google.com/web/updates/2017/04/headless-chrome) are a great place to start.