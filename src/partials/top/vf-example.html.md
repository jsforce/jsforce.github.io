---
---

### Example

```html
<apex:page docType="html-5.0" showHeader="false">
  <apex:includeScript value="{!URLFOR($Resource.JSforce)}" />
  <script>
    const conn = new jsforce.Connection({ accessToken: '{!$API.Session_Id}' });
    const res = await conn.query('SELECT Id, Name FROM Account');
    console.log(res);
  </script>
</apex:page>
```

