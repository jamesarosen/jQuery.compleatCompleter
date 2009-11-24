Designing an autocomplete framework is hard. **Really hard**. How do I know? I need one, and I need it to perform several different duties: as a search box, as the "TO:" field for a social network messaging form, and as a tag list. Each use case demands autocompletion, but they behave differently in subtle yet important ways.

First I tried [jQuery.autocomplete](http://docs.jquery.com/Plugins/Autocomplete), but that uses a silly custom response format. It's written in Javascript, why not use JSON? Additionally, you only get one completion per field. This solves one problem reasonably well.

James Smith's [jQuery.tokeninput](http://loopj.com/2009/04/25/jquery-plugin-tokenizing-autocomplete-text-entry/) uses sensible JSON and supports completing multiple items per field, but falls down on customizability. I want to change the class of the field during searching, not display the dropdown list with "Searching..." It's not as though there's *no* customizability. Indeed, I can customize all the various classes that get applied on events. But why would I? I can just give the top-level item a different class and use hierarchical CSS selectors to change styling. It also inserts some &lt;p&gt;s where I don't think they belong. Lastly, it *requires* each entry in the list to already have a database ID, since the ID list rather than the textual list is what is submitted in the form. This works fine for some fields, but poorly for tag list inputs. Again, the library solves one problem reasonably well.

Rafudu's [jQuery.tagbox](http://saynotofastfood.info/tagbox/examples/) does nifty things for tag inputs, but it doesn't have autocompletion. One more library that solves a narrowly-defined problem well.

=== The Dragon

So I started out on a new quest. My dragon: a customizable jQuery-based autocompletion framework that is well-suited for a variety of use cases. I like jumping in to TDD coding as much as the next fellow, but I'm not fluent in Javascript yet and I knew this was going to be a complicated problem.

=== The Townsfolk

`You are in TOWN. There are many TOWNSFOLK wandering about.`

I decided to start by talking to the townsfolk.

`talk with townsfolk`

`The TOWNSFOLK tell you many wondrous stories and feed you lots of STEW.`

I took their dragon stories and turned them into a a few major questions and a large number of related smaller questions that an adventurer -- sorry, an `ADVENTURER` -- should ask him- or herself. First, the big ones:

1. How many *things* can the user enter into the field? If it's a search box, only one. If it's a tag cloud, many. In some ways, the distinction is between **single** and **multiple**, though the **single** case often behaves like the very last item of the **multiple** case if there is a firm limit. Pressing <abbr title="Tab">`↹`</abbr> after entering text in a single-item autocomplete box should move focus to the next form object. On a multi-item box, it should start the next item.
1. What can users enter for each item? For our search box, **anything goes**. For the "TO:" field in a social networking site, it's probably **set-limited** to the user's friends. This distinction has surprisingly important behavioral differences. One important distinction is for **anything goes** fields, the user may very well want to see completions but fill in the rest of the field without help; automatically filling in the first completion would probably cause more pain than pleasure. For a **set-limited** field, on the other hand, it might very well make sense to automatically move focus to the first completion when results come back.
1. What do you use to delimit entries in a **multiple** list and how do you escape entries that have one or more of the delimiters in their text? (This is an area where `jquery.tokeninput` actually works quite well.) If <abbr title="Space">`⎵`</abbr> is a delimiter, are all the other Unicode whitespace characters (the three-em, the thin, the hair, the narrow no-break, etc.)?

And now the smaller ones, which all have the same form:

1. What happens when the user hits <abbr title="Return">`↵`</abbr> right after entering the field? After typing a few characters? After highlighting a completion? After inserting a completion? Do any of these answers change if the field in question is the only (non-hidden) field in the form? If it's the *last* (non-hidden) field in the form?
1. Similarly for <abbr title="Enter">`⌤`</abbr>, <abbr title="Tab">`↹`</abbr>, <abbr title="Semicolon">`;`</abbr>, <abbr title="Comma">`,`</abbr>, and <abbr title="Space">`⎵`</abbr>.

=== Fellow Adventurers

Talking to all those townsfolk meant eating lots of stew. Satiated and eager to slay my dragon, I decided to leave town.

`You are in TOWN. There are exits to the EAST and the NORTH.`

`go EAST`

`You are in the FOREST OF GYTHUB. There are many ADVENTURERS. Most of them are eating APPLES.`

`take apple`

`You already have an APPLE. Besides, that's not nice.`

I looked at other projects and asked about the kinds of things a user of the library (who is a programmer, as opposed to the end-user, who probably isn't) would want to customize. Here's the list so far:

* `url`
* `delay` - milliseconds to wait for a subsequent key-press before making an AJAX call, e.g. `300`
* `minChars` - minimum number of characters before making an AJAX call, e.g. `3`
* `maxItems` - maximum number of entries; `1` for a **single**-style list, `null` for an unlimited list, `n`∈ℕ for a **multiple** but limited list
* `affordDelete` - whether each item in the list comes with a little "x" for deleting; probably `true` by default iff `maxItems` &gt; 1
* `inReturnedItemsOnly` - **set-limited** versus **anything goes**
* `delimiters` - only if a **multiple** field, e.g. `/;\s+/`
* `programmaticDelimiter` - the delimiter that is inserted if the user selects an autocompletion rather than typing the whole entry and a delimiter manually, e.g. `';'`
* `groupStartChar` and `groupEndChar` - only if a **multiple** field, e.g. `"`
* `contentType` of the AJAX request, e.g. `'JSON'`
* `parseResponse()` - for use if you can't control the content returned from the server, a function to parse the response body into something of the form `[{name: "bar"}, {name: "baz"}]`
* `onFocus()`, `onStartItem()`, `onRequest()`, `onResponse()`, `onAutocompletionsDisplayed()`, `onAutocompletionFocus()`, `onAutocompletionBlur()`, `onInsertItem()`, `onCloseItem()`, `onAutocompletionsHidden()`, `onDeleteItem`, `onBlur()` - hooks, each bound to the autocomplete object so the library user has access to helpers, the relevant form, etc.

In addition to customizations, there are some core concepts that should be exposed as methods:

* `isLastItem()` - for autocomplete fields with a non-`null` `maxItems`, whether the user is currently working on the last allowed item
* `isLastVisibleField()` - whether the autocomplete field is the last (or only) visible field in a form

And some helper methods that users of the library can easily chain together to customize behavior:

* `startNewItem()`
* `closeItem()` - insert a delimiter
* `tentativelyCompleteText()` - fill in the rest of the text of the current item, but highlight it as *tentative*, leave the cursor in the middle of the item
* `insertHightlightedAutocompletion()` - insert the completion as a full item in the list, leave the cursor at the end of the item
* `propagateEventIfLastField()` and `propagateEvent()`

=== The Adventure

I'd love to have you join this adventure. I've [set up camp](http://github.com/jamesarosen/jQuery.compleatCompleter) in the `FOREST OF GYTHUB`. There probably aren't a lot of `GOLD PIECES` to be found with this dragon, but we might earn some experience.
