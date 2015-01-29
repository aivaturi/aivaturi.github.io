---
layout: post
title:  "Add getCSSCount command to Selenium"
date:   2009-12-30 15:35:00
categories: hacking selenium getcsscount
---

getXpathCount is a very useful command that Selenium devs provided. And once you started enjoying coding with it, you realized IE was throwing roadblocks since XPath is dog slow on it (what else is new?). So you look around and you're told to switch to CSS Selectors (or locators). Well, the very first thing you're looking for is the count equivalent for CSS and it is conveniently missing. So, I ended up hacking the Selenenium core to add the missing command - "getCSSCount".

BTW, I hacked the core that comes with the selenium-server.jar and I believe you can achieve the same result using user-extensions.js. Incidently, I understood the core better just by looking at the unjarred files, than reading the user-extensions doc - go figure. So just to be clear - my way is not the only way to do it and it works for me. Here are the steps:
 
- Unjar the selenium-server.jar.
- Inside the core/scripts directory you'll have to hack 2 files - selenium-api,js, selenium-browserbot.js
- In Selenium-api.js add this function just below getXpathCount function (or download the attached file & replace it).
{% highlight javascript %} 
Selenium.prototype.getCSSCount = function(locator) {
    /**
    * Returns the number of nodes that match the specified xpath, eg. "table" would give
    * the number of tables.
    * 
    * @param xpath the xpath expression to evaluate. do NOT wrap this expression in a 'count()' function; we will do that for you.
    * @return number the number of nodes that match the specified xpath
    */
    var result = this.browserbot.evaluateCSSCount(locator, this.browserbot.getDocument());
    return result;
}
{% endhighlight %}

- In selenium-browserbot.js add this function just below evaluateXpathCount function (or download the attached file & replace it).
{% highlight javascript %}
/**
 * Returns the number of css results.
 */
BrowserBot.prototype.evaluateCSSCount = function(css, inDocument) {
    var results = eval_css(css, inDocument);
    return results.length;
};
{% endhighlight %}

- Now, in iedoc-core.xml & iedoc.xml (in the core directory) add these lines after the getXpathCount block (or download the attached files & replace them).
{% highlight html %}
<function name="getCSSCount">
<return type="number">the number of nodes that match the specified CSS Locator</return>
<param name="xpath">the CSS Locator expression to evaluate. do NOT wrap this expression in a 'count()' function; we will do that for you.</param>
<comment>Returns the number of nodes that match the specified CSS Locator, eg. "table" would give
the number of tables.</comment>
</function>
{% endhighlight %}
 
- Now jar the whole file back using this command in the root directory - "jar cmf META-INF\MANIFEST.MF selenium-server.jar ." (note the dot at the end).
- The next step is to obviously expose this functionality through your driver - I use Perl & I'll provide you the example here. Take a look at it & modify your respective drivers, it is very easy. So get to your Selenium.pm file & add this function (or download the attached file & just replace your local copy):

{% highlight perl %}
=item $sel-E<gt>get_css_count($selector)
Returns the number of nodes that match the specified xpath, eg. "table" would givethe number of tables.
=over
$selector is the selector expression to evaluate. do NOT wrap this expression in a 'count()' function; we will do that for you.
=back
=over
Returns the number of nodes that match the specified selector
=back
=cut
sub get_css_count {
    my $self = shift;
    return $self->get_number("getCSSCount", @_);
}
{% endhighlight %} 
 
That is all that is required. Now you have getCSSCount available for your testing needs. For the Perl driver, here is how you'd use it:

{% highlight perl %}
my $val = $sel->get_css_count("table[class='tblList']>tbody>tr");
print "val: $val\n";
{% endhighlight %} 

From the above example, it'll give me the number of rows in a table of class tblList. Note that you don't have to use the "css=" prefix.
