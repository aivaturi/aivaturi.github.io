---
layout: post
title:  "Hacking Selenium to improve its performance on IE"
date:   2010-02-03 16:14:00
categories: selenium performance IE
---

TL;DR of the post below; You have to perform all these steps & in some instances, you can squeeze out better performance from other browsers too:

1. Switch to "javascript-xpath". The default ajaxslt is unbearably slow when it comes to IE. I don't even know why it is still distributed with Selenium.
2. Inject javascript-xpath in to the DOM of your app under test - each time it changes. In our case, processing speed jumped at least 10 times by doing this. You might have reservations about modifying your app under test but this step is for all practical purposes - harmless & javascript-xpath remains dormant in browsers where xpath support is already built in (read - any browser other that IE).
3. Offload repetitive processing to javascript through user extensions & use "context nodes" where possible - specifically where you have to read table listing. More details in the description below.
4. This is highly optional but convert your ids to xpath. Yes, you heard that right - xpath look up (with the first 2 steps above) is at least 3 times faster than plain id lookup in IE.

All the above will practically bring your IE performance really close to that of Firefox. Now for the original post with details:

Here is some info on what we did to improve the IE performance with regards to Selenium. Before we get started, you have to familiarize yourself with a few key related technologies:
 
- XPath - [http://www.w3.org/TR/xpath/][xpath]
- CSS Selectors - [http://www.w3.org/TR/css3-selectors/][css]
- DOM - [http://www.w3.org/DOM/][dom]
- JSON - [http://www.json.org/][json]
 
Of course, you also have to know what Selenium is, how it works & a basic idea of its architecture to understand the problem & how to solve it. All major browsers in the market (FF, Chrome, Safari) except IE include XPath engines in their browsers. This makes is extremely simple to query a web element on the page directly through JavaScript. For an intro to the API, please read this document - [https://developer.mozilla.org/En/Introduction_to_using_XPath_in_JavaScript][xpath-api]. Selenium (apart from other locators) uses XPath to help you locate elements on a web page. But since IE does not have a native XPath engine, it is actually implemented as a JavaSript Library by 3rd party. Google has its own implementation known as ajaxslt, which is the default in Selenium for XPath processing if native support is unavailable. But that is where the problem starts.
 
##### The Problem

Selenium launches the app under test browser window in two modes - single window & multi window. Default is always multi window wherein, the first one is the driver window which processes all the commands & the second window is the actual app under test. In single window mode, the app under test is loaded in to the lower frame, which means the app under test cannot be frame busting - meaning the app under test should not have any frames or popup windows. Since our app is frame busting, we have to use the default mode - multi window. When you send a command to evaluate an XPath query, the driver window uses the DOM handle of the app under test & runs that query by traversing the DOM. 

In FF, this is amazingly fast as it has a native support for XPath, but on IE the driver window uses the external XPath JavaScript library to do the same. But, the evaluation gets progressively slow on large set of elements due to 2 main reasons:

1. Extremely slow JavaScript engine of IE 
2. Constant chatter between the driver window & the app under test window, which was addressed recently by the Selenium devs (check out the latest code from http://code.google.com/p/selenium). The driver window ends up doing a DOM traversal on the app under test after processing the XPath query. 

And on top of those, there was another problem - ajaxslt was itself very slow. Fortunately, there is another library called [javascript-xpath][js-xpath] and in Selenium you can switch to that library to improve the performance of your XPath processing. Inside Selenium, it didn't really help much but outside of Selenium the processing was blazingly fast. A lot of discussion ensued with the developers to convince them that this was in no way a usable solution. You can read all about it here - http://groups.google.com/group/selenium-developers/browse_thread/thread/160a92fcacb12a62 and subsequently a bug was filed with the relevant info http://code.google.com/p/selenium/issues/detail?id=307.
 
##### CSS Selectors to rescue?

Our automation framework depends on Selenium to interact with the web pages (Juniper SSL VPN admin pages). It has various "get" & "set" methods to read & write values to those web pages. We implemented a convenient method in our framework called "table-list", which reads the list of items listed in a specific table format and was widely used across all the admin pages. Reads are always costly as individual pages can be huge for e.g. Auth Servers page (linked to the bug above) was taking about 25 minutes to finish listing the auth servers with the ajaxslt library. For comparison, on FF it was taking close to 4 seconds. So after digging in the forums for a while, CSS selectors was offered as a faster alternative. 

But CSS Selectors themselves weren't that blazingly fast. When we re-implemented that "table-list" method using CSS Selectors, the time of execution dropped down to 6 minutes, but still, it was really slow. So, we started looking for other alternatives and in one of those experiments we realized that all this chatter between the two browser windows was slowing down thing & we could probably improve the performance a bit if we had javascript-xpath library on the app under test window. We did exactly that - once the app under test window is loaded, we modify the DOM & inject the javascript-xpath library. This actually sped up over all performance on IVE Admin pages from login onwards and our "table-list" dropped down to just over 4 minutes. This still was not acceptable.
 
##### "Context nodes" & injecting javascript-xpath

If you have ever worked with libxml2 XML processing library, you'll know about a little nugget known as "context" nodes. In a structured document like XML & XHTML, you can traverse the document as a tree & if you want to do repetitive processing on a set of nodes, you chose the parent of those nodes & make it a temporary root node and this node is known as "context" node. That means all your further processing will work based on this particular context & you don't have to traverse the whole document for each processing of a query. Well there should be something similar in XPath too right? Yes of coursei, [there was][context-node].

If you look at the test document of the Auth Servers page, attached to the bug filed with Selenium, there are over 800 tables in that page (don't ask me why - I still don't know). So whenever we ran some thing like //table[@class='tblList']  it'd go through the whole document (with all those 800 tables) & do a search for that table and then drill down for eventual table cell where our content resides. With context nodes (outside of selenium), the whole "table-list" operation dropped down to 900 milliseconds, since you're not searching through 800 tables  anymore. There was a hurdle though; Selenium does not support returning object back to the client driver - only data. That means, we could never get the context node back from Selenium. So, we decided to completely off load "table-list" processing to JavaScript using the user-extensions feature of Selenium. 

Using user-extensions, you can extend Selenium & create your own commands. And thus the getList command was born, which does exactly the same as "table-list", only it was implemented in JavaScript & not Perl. We used JSON to build our data structure, serialize it & send it over the wire to Perl Client driver. The execution time dropped down to 2 seconds for that command on IE - mission accomplished! On a side note - in FF this same method will finish execution in milliseconds.
 
Now, that we solved the issue of listing the content, we feared that the general "reads" on IE for input elements with no ID, would be slow too and indeed it was. Our IVE web page templates didn't have IDs for elements - only names & values. So when you try to read the input elements with name/value attribute, the DOM traversal was horrendously slow in IE. So the solution was to inject javascript-xpath at runtime on every single page & convert the element names to XPath on the fly & process them. Well, this brought down the processing time of each lookup from over 15 seconds to milliseconds on IE. All these changes dramatically improved the performance & brought it very close to the performance of Selenium on FF. 

At this point, you might ask if it is wise to modify app under test? Well, it is a judgement call & in our case we're modifying app under test to inject the javascript-xpath to improve the performance of XPath locators on IE. javascript-xpath does not interfere or modify any thing else on your document's DOM and is in fact dormant in browsers which have native support for XPath. As I see it, it was really not an issue. 

[xpath]: http://www.w3.org/TR/xpath/
[css]: http://www.w3.org/TR/css3-selectors/
[dom]: http://www.w3.org/DOM/
[json]: http://www.json.org/
[xpath-api]: https://developer.mozilla.org/En/Introduction_to_using_XPath_in_JavaScript
[js-xpath]: http://coderepos.org/share/wiki/JavaScript-XPath
[context-node]: http://groups.google.com/group/comp.lang.javascript/browse_thread/thread/a59ce20639c74ba1/a9d9f53e88e5ebb5
