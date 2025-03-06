---
layout: post
title: Obsidian quick Tip - UUIDs
category:
  - "obsidian"
tags:
  - "dataview"
  - "templater"
  - "javascript"
  - "obsidian"
excerpt_separator: "<!--more-->"
asset_path: "/assets/images/blog/{{ page.title }}"
image: ""
---

Here's a quick tip / best practice I am going to implement from now on in any of my Obsidian based projects. After struggling with a problem to query all associated files with one "master" file I took my own advice. If I want to use the `DataviewAPI`  / [Dataview Plugin for Obsidian](https://github.com/blacksmithgu/obsidian-dataview) to query my Obsidian notes in an SQL-like syntax I also need to treat my notes like a (SQL or any other) database would treat it's data.

<!--more-->

# The problem

To give more context, here's a quick overview of the struggle I've faced.
As you might be aware I have a rather elaborate setup to track my workouts. The notes I use are:

1. A note called "🍏 Health" in which I track the current strength cycle. It features a metadata fild called `trainingplan::` with a link to the current training plan --> `trainingplan:: [[III 531 cycle]]`
2. The training plan note itself which features numerous metadata fields and acts as the source for individual workouts
3. A daily note called "Physical practice" (also tagged `#physicalpractice`) which I keep alongside my daily notes to track my - you guessed it - physical practice for the day. If I ever do a strength based workout from the current cycle I do so by inserting a little template snippet in this note. The template not only features the work to be done but also a reference to the source aka the current training plan in form of an inline metadata field `trainingplan::  [[III 531 cycle]]`  

I now wanted to write a query which returns every "physical practice" page, which links back to my "III 531 cycle" page. 
So something like this:

First I query the current training plan

```javascript
let healthpage = DataviewAPI.page("🍏 Health");
let trainingplan = DataviewAPI.page(healthpage.trainingplan.path);
```

Now that we have our current trainingplan, get all pages with the hashtag "physicalpractice" but filter them and return only those which paths match with the path of the current training plan.

```javascript
let allPages = DataviewAPI.pages('#physicalpractice').where(m => m.trainingplan.path === trainingplan.file.path);
```

Should work right? Wrong. This is the error I got. 
```
VM7901:1 Uncaught TypeError: Cannot read properties of undefined (reading 'path')
    at <anonymous>:1:35
    at Array.filter (<anonymous>)
    at Proxy.where (plugin:dataview:8189:39)
    at <anonymous>:1:10
```

The problem seemed to be the link from the `#physicalpractice` files. Somehow (to this day I don't know why) I couldn't access the `.path`-property.
I even tried some hardcoded stuff:

```javascript
let testpages = DataviewAPI.pages('#physicalpractice');
testpage[33].trainingplan.path === trainingplan.file.path
// --> true
```

Apparantely the comparison (and `.path` property) works this way but not in the `.where()` function. 

# Solution

I thought hard about how to get the property right in order to compare paths. But after a good nights sleep I've remebered a takeaway from [my post about how I track my training]({% post_url 2023-11-03-obsidian-training-plan %}):

```
Try to treat files with metadata more like database entries and think hard about which values you track with metadata. 
Sticking to database normalization helped me a lot (although it obviously cant be applied 100%).
```

So - why not start to use a `UUID`?
Implementation ist pretty straightforward with [`crypto.randomUUID()`](https://developer.mozilla.org/en-US/docs/Web/API/Crypto/randomUUID).

After generating a UUID and adding a new metadatafiled simply called `uuid` to my trainingplan page I just needed to update my `#physicalpractice` notes with another metadata field which I called `tpuuid::`. In database terms this would be your foreign key (I ponder to rename it to `uuid_fk` or something else to have some kind of standard way on how to approach this).

After update all of my notes the query looks like this and works perfectly:

```javascript
DataviewAPI.pages('#physicalpractice').where(m => m.tpuuid === trainingplan.tpuuid)
```

It's more readable and probably a bit faster - allthough speed isn't something I worry about when querying my notes (not yet..).

This left me only to update my templates.
I've added this little snippet to my template for my `#physicalpractice` notes

```javascript
const tpUUID = trainingplan.UUID
tpUUID:: <% tpUUID %>
```

so that from now on every template holds the corresponding uuid from the training plan.
I've also adjusted my templates to generate new training cycles to now feature a uuid:

```javascript
const uuid = crypto.randomUUID();
uuid:: <% uuid %>
```

# Conclusion

From now on I will probably rely more heavily on the UUID concept at least for notes which feature some sort of "main" characteristic - that is notes which are at the root of a project, theme, area etc. Those will need to be identified more often so the will have a uuid. Sub sites therefore will then back-reference to the main site. 
This obviously is only one part of the relation and I'm already thinking about completing it to make it more future proof.
I'll keep you posted.
