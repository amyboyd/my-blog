---
layout: post
title:  Validate custom HTML rules with htmlcheck
tags: htmlcheck html nodejs javascript
permalink: /:year/:month/:day/:title
excerpt: htmlcheck is an open-source package I wrote for Node that can quickly check HTML files for certain DOM structures and requirements.
---

Often when building a frontend, there will be certain rules that the HTML should follow, either
dictated by your CSS, by a style guide, or business. For example, here's a few HTML rules I've
seen recently:

* `<button>` elements should have an icon inside them.
* `<img>` elements should all have an 'alt' text, and `<button>` elements should have an 'aria-label'.
* When using Angular, you need to ensure all `<a>` and `<button` elements go somewhere or do
  something, so they must have one of 'href', 'ng-href' or 'ng-click' set.

htmlcheck
---------

htmlcheck is an open-source package I wrote for Node that can quickly check HTML files for certain
DOM structures and requirements.

Examples
--------

Check that every `<button>` and `<a class="button">` have an icon and an 'aria-label'.

<pre>
htmlcheck
    .forElements('button')
    .forElements('a.button')
    .inDir('src/templates')
    .test(function(button) {
        const icons = button.querySelectorAll('i.icon, span.icon');
        if (icons.length !== 1) {
            throw `Button with text "${button.textContent.trim()}" has ${icons.length} icons, should have one`;
        }
    })
    .test(function(button) {
        if (!button.hasAttribute('aria-label')) {
            throw `Button with text "${button.textContent.trim()}" does not have aria-label attribute; add this for screen readers`;
        }
    });
</pre>

Check that every `<button>` and `<a>` have one of 'href', 'ng-href', or 'ng-click'.

<pre>
htmlcheck
    .forElements('a')
    .forElements('button')
    .inGlob('src/**/*.html')
    .test(function(link) {
        function isAttributeOk(attribute) {
            const value = link.getAttribute(attribute);
            return typeof value === "string" && value.length > 0;
        };

        const isOk = isAttributeOk('href') || isAttributeOk('ng-href') || isAttributeOk('ng-click');

        if (!isOk) {
            throw `Link with text "${link.textContent.trim()}" has no action; add one of "href", "ng-href" or "ng-click"`;
        }
    });
</pre>

Get started
-----------

Install htmlcheck from NPM in the usual way: `npm install htmlcheck --save`

Here are some links with more information:

* <a href="https://www.npmjs.com/package/htmlcheck">NPM package page</a>
* <a href="https://github.com/amyboyd/htmlcheck">GitHub page with source code</a>
* <a href="https://github.com/amyboyd/htmlcheck/issues">Report bugs and issues here</a>
