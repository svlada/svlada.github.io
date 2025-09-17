---
title: Proxying cross-domain requests using Symfony 2
description: Proxy Ajax requests, Curl and Symfony 2
date: 2013-12-23
tags:
  - symfony
  - curl
layout: layouts/post.njk
permalink: "proxy-ajax-requests-curl-and-symfony-2/index.html"
---
This article explains how to initiate cross-domain requests through a proxy using cURL and Symfony 2.

👉 Jump to [#source-code](#source-code).

The usual scenario looks like this:
1. The client sends an AJAX request to your server.
2. Your server forwards the request to an external (remote) server.
3. The server waits for the response from the remote server.
4. The response is parsed and processed.
5. The processed response is sent back to the client.

As a first step, check if the client request is an XmlHttpRequest. Symfony 2 provides a built-in method for this:
 
```php
 $request->isXmlHttpRequest()
```
 
### Step 1: Client Code

The following steps describe how to create an AJAX request.
1. Specify the REST URL on the server for handling cross-domain AJAX requests: `url: "{{ path('_ajaxProxy') }}"`
2. Wrap the request data.

You'll need some data to create a cURL request on the server side.

For this example, send an object with the following properties to your server:
- **url** → the remote server endpoint to call
- **method** → the HTTP method (e.g., GET, POST)
- **params** → request parameters or payload data
 
```js
restUrl: "external-api-url", // Your target url on remote server
method: "POST", // Type of request you want to issue to remote server
params: { 
    action: "getFriendsList" // Parameters you are sending to remote server
}
```
 
### Step 2: Forward the Request to the Remote Server

This step is a bit more involved.

When the client request reaches your server, the first task is to parse and validate the request data.

To recap, your server needs the following information from the client:
 
```php
$restUrl = $request->request->get('restUrl'); // Your target url on remote server
$method = $request->request->get('method'); // Type of request you want to issue to remote server
$params = $request->request->get('params'); // Parameters you are sending to remote server
$contentType = $request->request->get('contentType'); // Content-type
```
 
### Step 3: Create the cURL Request

On the server side, create a cURL request and set its parameters using the values received from the client.

```php
// Initialize curl handle
$ch = curl_init();  
// Set request url
curl_setopt($ch, CURLOPT_URL, $restUrl); 
// TRUE to include the header in the output.
curl_setopt($ch, CURLOPT_HEADER, true);  
// A custom request method to use instead of "GET" or "HEAD" when doing a HTTP request.  
curl_setopt($ch, CURLOPT_CUSTOMREQUEST, $method); 
if ($params != null) {
    // The full data to post in a HTTP "POST" operation.
    curl_setopt($ch, CURLOPT_POSTFIELDS, http_build_query($params));
}
curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
curl_setopt($ch, CURLOPT_FOLLOWLOCATION, true);
```
 
I strongly recommend reading the [PHP manual on curl_setopt](http://php.net/manual/en/function.curl-setopt.php) to get familiar with the available cURL configuration options.

At this point, you have all the necessary data to send a request to the remote server. After executing the cURL handle, the response from the remote server will be stored in the `$response` variable.
 
```php
$response = curl_exec($ch);
curl_close($ch);
```
 
You now have a basic setup. Next, let's add cookie support to the AJAX proxy.

Why do we need cookies?

In my case, I had to integrate a Symfony 2 application with a WordPress portal. Authentication was handled on the WordPress side, while the Symfony application relied on WordPress cookies to authorize users for accessing protected features.

In production, both applications shared the same domain. But in the test environment, things were more complicated—different domains, ports, and other inconsistencies made the setup messy.

Where's the problem?

The Symfony 2 application needed to consume protected backend functionality from WordPress. Since much of the data was personalized and fetched via AJAX calls, cookies were required to maintain authentication and authorization.
 
### Getting Cookies from Symfony 2 and Passing Them to cURL

To forward cookies from a Symfony 2 request to a remote server, you need to:
1. Extract cookies from the Symfony request object.
2. Format them into a string that cURL understands.
3. Set them on the cURL handle.
 
```php
// Get all cookies from Symfony request object
$requestCookies = $request->cookies->all(); 
// Prepare and set multiple cookies to curl handle
$cookieArray = array();
foreach ($requestCookies as $cookieName => $cookieValue) {
    $cookieArray[] = "{$cookieName}={$cookieValue}";
}
// Be sure to set whitespace after '; ' when creating cookie string
$cookie_string = implode('; ', $cookieArray);
curl_setopt($ch, CURLOPT_COOKIE, $cookie_string);
```
 
### Getting Cookies from cURL Response and Passing Them to Symfony Response

When you configure cURL with `CURLOPT_HEADER = true`, the response will include both headers and body. From there, you can parse the Set-Cookie headers and attach them to your Symfony response.
 
```php
// Get header and response data from curl response
list($headers, $response) = explode("\r\n\r\n",$response,2);
// We are using regex to parse cookies from curl response
preg_match_all('/Set-Cookie: (.*)\b/', $headers, $cookies);
// Store cookies
$cookies = $cookies[1];
```

Raw cookies are parsed into an array, then each entry is converted to a Symfony Cookie and added to the response headers.
 
```php
foreach($cookies as $rawCookie) {
    $cookie = \Symfony\Component\BrowserKit\Cookie::fromString($rawCookie);
    $value = $cookie->getValue();
        if (!empty($value)) {
            $value = str_replace(' ', '+', $value);
        }
    $customCookie = new Cookie($cookie->getName(), $value, 
        $cookie->getExpiresTime()==null?0:$cookie->getExpiresTime(), $cookie->getPath());
    $response->headers->setCookie($customCookie);
    }
```
 
⚠️ **Final Notice:**
Be sure to close and store session data before initializing the cURL handle—otherwise, you may run into a deadlock.

 
```php
session_write_close();
$ch = curl_init();
```
 
What happens if you forget to close the session?

When PHP uses the default session handler, it locks the session file for the duration of the request. If S1 (a PHP script) starts a cURL POST to S2 (another PHP script) on the same server and both share the same session:
- S1 has the session locked.
- S2 can't start (it tries to open the same session and blocks).
- S1 is waiting for S2's response… which never comes because S2 is blocked by the lock.
- Result: deadlock.

Fix: call `session_write_close();` before you open the cURL handle so other requests can read the session.

### Complete source code listing for the AJAX proxy (Symfony 2)
 
 **Client sample code**
```js
function sendAjaxRequest(){
    $.ajax({
        type: "POST",
        dataType: 'json',
        url: "{{ path('_ajaxProxy') }}",
        data: {
        restUrl : "/external-api-url/",
        method: 'POST',
            params: {
            action: "getFriendList"
            }
        },
        success: function(data, textStatus, jqXHR) {
            ...
    }
});
}
```
 
**Server proxy code**

```php
namespace Proxy\Bundle\ProxyBundle\Controller;

use Sensio\Bundle\FrameworkExtraBundle\Configuration\Template;
use Symfony\Bundle\FrameworkBundle\Controller\Controller;
use Sensio\Bundle\FrameworkExtraBundle\Configuration\Method;
use Sensio\Bundle\FrameworkExtraBundle\Configuration\Route;
use Symfony\Component\HttpFoundation\Response;
use Symfony\Component\HttpFoundation\Request;
use Symfony\Component\HttpFoundation\Cookie;

class AjaxProxyController extends Controller {
/**
    * @Route("/r/ajax/proxy", name="_ajaxProxy")
    * @Template()
    */
public function proxyAction(Request $request)
{
    // Forbid every request but jquery's XHR
    if (!$request->isXmlHttpRequest()) {// isn't it an Ajax request?
        return new Response('', 404, 
                            array('Content-Type' => 'application/json'));
    }
    
    $restUrl = $request->request->get('restUrl');
    $method = $request->request->get('method');
    $params = $request->request->get('params');
    $contentType = $request->request->get('contentType');
    
    if ($contentType == null) {
        $contentType = 'application/json';
    }

    if ($restUrl == null || $method == null || 
                        !in_array($method, array('GET', 'POST', 'DELETE'))) {
        return new Response('', 404, array('Content-Type' => $contentType));
    }
    
    session_write_close();
    $ch = curl_init();
    curl_setopt($ch, CURLOPT_URL, $restUrl);
    curl_setopt($ch, CURLOPT_HEADER, true);
    curl_setopt($ch, CURLOPT_CUSTOMREQUEST, $method);
    if ($params != null) {
        curl_setopt($ch, CURLOPT_POSTFIELDS, http_build_query($params));
    }
    curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
    curl_setopt($ch, CURLOPT_FOLLOWLOCATION, true);
    
    $requestCookies = $request->cookies->all();
    
    $cookieArray = array();
    foreach ($requestCookies as $cookieName => $cookieValue) {
        $cookieArray[] = "{$cookieName}={$cookieValue}";
    }
    $cookie_string = implode('; ', $cookieArray);
    curl_setopt($ch, CURLOPT_COOKIE, $cookie_string);
    
    $response = curl_exec($ch);
    curl_close($ch);
    
    list($headers, $response) = explode("\r\n\r\n",$response,2);
    preg_match_all('/Set-Cookie: (.*)\b/', $headers, $cookies);
    $cookies = $cookies[1];
        
    if ($response === false) {
        return new Response('', 404, array('Content-Type' => $contentType));
    } else {
        $response = new Response($response, 200, 
                                            array('Content-Type' => $contentType));
        foreach($cookies as $rawCookie) {
            $cookie = \Symfony\Component\BrowserKit\Cookie::fromString($rawCookie);
            $value = $cookie->getValue();
            if (!empty($value)) {
                $value = str_replace(' ', '+', $value);
            }
            $customCookie = new Cookie($cookie->getName(), $value, $cookie->getExpiresTime()==null?0:$cookie->getExpiresTime(), $cookie->getPath());
            $response->headers->setCookie($customCookie);
        }
        return $response;
    }
}
}
```

**Read more about this topic:**

- http://developer.yahoo.com/javascript/howto-proxy.html
