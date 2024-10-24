Doing a Userscript to Block X/Twitter Ads
===========================

Recently I've noticed twitter has a lot more ads than before. 

AND I'm also currently on a kick of writing userscripts for everything. They're just so versatile and easy to make. You can make these big tech websites feel like yours again. Feel like you have control.

Since they are small javascript snippets, LLMs do a really good job of generating code for it + the browser console is a pretty good interpreter. So this morning I decided to make this script to block twitter ads.


Download "ViolentMonkey" > Click Extension > Click + new > Paste > Ctrl+S to save

> (If you use it in portuguese or other languages, you might have to change the `.includes('Ad')` to `.includes('Anúncio')`. But that's it.)

```javascript
// ==UserScript==
// @name         Hide X Ads
// @namespace    http://tampermonkey.net/
// @version      0.1
// @description  Hides ads on X home timeline
// @author       Tonio + GPT-4o
// @match        https://X.com/*
// @grant        none
// ==/UserScript==

(function() {
    'use strict';

    function hideAds() {
        let elements = document.querySelector('[aria-label="Timeline: Your Home Timeline"]')
                       .querySelectorAll('[data-testid="cellInnerDiv"]'); // selector for tweet divs

        for (let element of elements) {
            let spans = element.getElementsByTagName('span');
            for (let span of spans) {
                if (span.textContent.includes('Ad')) {
                    element.style.display = 'none';
                }
            }
        }
    }

    hideAds();
    setInterval(hideAds, 10000); // Repeat every 10 seconds (not perfect but it works /shrug)
})();
```
