# HTTP Errors

The GENI API uses the following HTTP error codes:


Error Code | Meaning
---------- | -------
200 | Okay -- Request is valid.
201 | Created -- Successful creation of an entity.
204 | No Content -- Requests that don't return any content when they succeed. 
400 | Bad Request -- Request is invalid.
401 | Unauthorized -- User is not logged in or session is expired or your API key is not authenticated. 
403 | Forbidden -- You are not authorized to perform this request.
404 | Not Found -- The specified resource could not be found.
405 | Method Not Allowed -- You tried to access a resource with an invalid method.
409 | Conflict - Object is not in the expected state (e.g. trying to activate an account that was already activated). 
415 | Unsupported Media Type -- Content-Type header isn't acceptable. 
498 | ArcGIS Token Expired/Invalid -- Returned by the **GET /auth/login** method only. 
500 | Internal Server Error -- Framework failure of unhandled exception.
502 | Bad Gateway -- Seen in the test/production environments if the app server is not responding to the proxy server.
