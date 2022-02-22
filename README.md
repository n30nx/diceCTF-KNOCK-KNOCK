# diceCTF-KNOCK-KNOCK
Writeup of the knock knock from diceCTF 2022

This challenge seemed very easy to me because when I looked at the source code I
noticed that the formatting in this.secret variable was broken.

```js
class Database {
  constructor() {
    this.notes = [];
    this.secret = `secret-${crypto.randomUUID}`;
  }
```

And when creating a note it used the length of this.notes variable, and length's
hashed form, that has been created with generateToken function inside of Database

```js
  createNote({ data }) {
    const id = this.notes.length;
    this.notes.push(data);
    return {
      id,
      token: this.generateToken(id),
    };
  }
```


Then I noticed that, after declaring a database constant, it immediately created
a note with the flag in it. So the id of the flag had to be zero. I just went
and used the generateToken function on my computer with the id zero and with
secret 'secret-${crypto.randomUUID}'.

```js
  generateToken(id) {
    return crypto
      .createHmac('sha256', this.secret)
      .update(id.toString())
      .digest('hex');
  }
```

Then it didn't work. I was so sure of myself and I couldn't believe it. After
some time I finally decided to look at to the Dockerfile. And I saw that the
server used node:17.4.0-buster-slim, and I had node 12 in my computer. I
installed the node 17.4.0 and tried to run the generateToken function with same
parameters, and it worked.

The whole code:

```js
const crypto = require('crypto');

class Database {
  constructor() {
    this.notes = [];
    this.secret = `secret-${crypto.randomUUID}`;
  }

  createNote({ data }) {
    const id = this.notes.length;
    this.notes.push(data);
    return {
      id,
      token: this.generateToken(id),
    };
  }

  getNote({ id, token }) {
    if (token !== this.generateToken(id)) return { error: 'invalid token' };
    if (id >= this.notes.length) return { error: 'note not found' };
    return { data: this.notes[id] };
  }

  generateToken(id) {
    return crypto
      .createHmac('sha256', this.secret)
      .update(id.toString())
      .digest('hex');
  }
}

const db = new Database();
db.createNote({ data: process.env.FLAG });

const express = require('express');
const app = express();

app.use(express.urlencoded({ extended: false }));
app.use(express.static('public'));

app.post('/create', (req, res) => {
  const data = req.body.data ?? 'no data provided.';
  const { id, token } = db.createNote({ data: data.toString() });
  res.redirect(`/note?id=${id}&token=${token}`);
});

app.get('/note', (req, res) => {
  const { id, token } = req.query;
  const note = db.getNote({
    id: parseInt(id ?? '-1'),
    token: (token ?? '').toString(),
  });
  if (note.error) {
    res.send(note.error);
  } else {
    res.send(note.data);
  }
});

app.listen(3000, () => {
  console.log('listening on port 3000');
});
```
