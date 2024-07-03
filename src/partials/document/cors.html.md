## CORS

If you're getting CORS-related errors when using jsforce in the browser it might be that the API being used doesn't support CORS, see:
https://help.salesforce.com/s/articleView?id=sf.extend_code_cors.htm&type=5

For those cases you'll need to use the `jsforce-ajax-proxy` proxy server, check the README for setup instructions:
https://github.com/jsforce/jsforce-ajax-proxy/
