---
---

### Setup

```shell
$ npm install jsforce
```

### Example

```javascript
const jsforce = require('jsforce');

const conn = new jsforce.Connection();
await conn.login('username@domain.com', 'password')

const res = await conn.query('SELECT Id, Name FROM Account')
console.log(res)
```
