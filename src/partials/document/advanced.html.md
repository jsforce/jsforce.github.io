---
---

## Advanced Topics

### Record Stream Pipeline

Record stream is a stream system which regards records in its stream, similar to Node.js's standard readable/writable streams.

Query object - usually returned by `Connection.query(soql)` / `SObject.find(conditions, fields)` methods -
is considered as `InputRecordStream` which emits event `record` when received record from server.

Batch object - usually returned by `Bulk-Job.createBatch()` / `Bulk.load(sobjectType, operation, input)` / `SObject.bulkload(operation, input)` methods -
is considered as `OutputRecordStream` and have `send()` and `end()` method to accept incoming record.

You can use `InputRecordStream.pipe(outputRecordStream)` to pipe record stream.

RecordStream can be converted to usual Node.js's stream object by calling `RecordStream.stream()` method.

By default (and only currently) records are serialized to CSV string.


#### Piping Query Record Stream to Batch Record Stream

The idea of record stream pipeline is the base of bulk operation for queried record.
For example, the same process of `Query.destroy()` can be expressed as following:


```javascript
//
// This is much more complex version of Query.destroy().
//
const Account = conn.sobject('Account');
Account.find({ CreatedDate: jsforce.Date.TODAY })
  .pipe(Account.deleteBulk())
  .on('response', (rets) => {
    console.log(rets)
  })
  .on('error', (err) => {
    console.error(err)
  });
```

And `Query.update(mapping)` can be expressed as following:

```javascript
//
// This is much more complex version of Query.update().
//
const Opp = conn.sobject('Opportunity');
Opp.find(
    { "Account.Id" : "0010500000gYx35AAC" },
    { Id: 1, Name: 1, "Account.Name": 1 }
  )
  .pipe(jsforce.RecordStream.map((r) => {
    return { Id: r.Id, Name: `${r.Account.Name} - ${r.Name}` };
  }))
  .pipe(Opp.updateBulk())
  .on('response', (rets) => {
    console.log(rets)
  })
  .on('error', (err) => {
    console.error(err)
  });
```

Following is an example using `Query.stream()` (inherited `RecordStream.stream()`) to convert a record stream to a Node.js stream,
in order to export all queried records to CSV file.

```javascript
const csvFileOut = require('fs').createWriteStream('path/to/Account.csv');
conn.query("SELECT Id, Name, Type, BillingState, BillingCity, BillingStreet FROM Account")
    .stream() // Convert to Node.js's usual readable stream.
    .pipe(csvFileOut);
```

#### Record Stream Filtering / Mapping

You can also filter / map queried records to output record stream.
Static functions like `InputRecordStream.map(mappingFn)` and `InputRecordStream.filter(filterFn)` create a record stream
which accepts records from upstream and pass to downstream, applying given filtering / mapping function.

```javascript
//
// Write down Contact records to CSV, with header name converted.
//
conn.sobject('Contact')
    .find({}, { Id: 1, Name: 1 })
    .map((r) => {
      return { ID: r.Id, FULL_NAME: r.Name };
    })
    .stream().pipe(fs.createWriteStream("Contact.csv"));
//
// Write down Lead records to CSV file,
// eliminating duplicated entry with same email address.
//
const emails = {};
conn.sobject('Lead')
    .find({}, { Id: 1, Name: 1, Company: 1, Email: 1 })
    .filter((r) => {
      const dup = emails[r.Email];
      if (!dup) { emails[r.Email] = true; }
      return !dup;
    })
    .stream().pipe(fs.createWriteStream("Lead.csv"));
```

Here is much lower level code to achieve the same result using `InputRecordStream.pipe()`.


```javascript
//
// Write down Contact records to CSV, with header name converted.
//
conn.sobject('Contact')
    .find({}, { Id: 1, Name: 1 })
    .pipe(jsforce.RecordStream.map((r) => {
      return { ID: r.Id, FULL_NAME: r.Name };
    }))
    .stream().pipe(fs.createWriteStream("Contact.csv"));
//
// Write down Lead records to CSV file,
// eliminating duplicated entry with same email address.
//
const emails = {};
conn.sobject('Lead')
    .find({}, { Id: 1, Name: 1, Company: 1, Email: 1 })
    .pipe(jsforce.RecordStream.filter((r) => {
      const dup = emails[r.Email];
      if (!dup) { emails[r.Email] = true; }
      return !dup;
    }))
    .stream().pipe(fs.createWriteStream("Lead.csv"));
```

#### Example: Data Migration

By using record stream pipeline, you can achieve data migration in a simple code.

```javascript
//
// Connection for org which migrating data from
//
const conn1 = new jsforce.Connection({
  // ...
});
//
// Connection for org which migrating data to
//
const conn2 = new jsforce.Connection({
  // ...
});
//
// Get query record stream from Connetin #1
// and pipe it to batch record stream from connection #2
//
const query = await conn1.query("SELECT Id, Name, Type, BillingState, BillingCity, BillingStreet FROM Account");
const job = conn2.bulk.createJob("Account", "insert");
const batch = job.createBatch();
query.pipe(batch);
batch.on('queue', () => {
  jobId = job.id;
  batchId = batch.id;
  //...
})
```
