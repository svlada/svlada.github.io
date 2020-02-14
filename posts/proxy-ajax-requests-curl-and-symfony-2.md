---
title: Proxy Ajax requests, Curl and Symfony 2
description: This is introductory RequireJS tutorial in RequireJs series.
date: 2013-12-23
tags:
  - symfony
  - curl
layout: layouts/post.njk
permalink: "proxy-ajax-requests-curl-and-symfony-2/index.html"
---
This article provide information on how to initiate cross-domain requests through the proxy using Curl and Symfony 2.
 
Jump to the [#source-code](#source-code).
 
The usual scenario looks like this:
 
 1. The client sends ajax request to the server
 2. Your server forwards request to external/remote server
 3. Waiting on response from a remote server
 4. Parse and process response from remote server
 5. Send response back tothe  lient
 
First, check if the client request is XmlHttpRequest. This can be done using Symfony 2 built-in method:
 
```php
 $request->isXmlHttpRequest()
```
 
### STEP 1: Client code
 
The following are the steps for creating an Ajax request.
 
1. Specify rest URL on the server for handling cross-domain ajax requests.`
 url: "{{ path('_ajaxProxy') }}"`
2. Wrap request data
 
We need some data to create a curl request on the server-side.

For this example, you need to send an object with the following properties to your server.
 
```js
restUrl: "external-api-url", // Your target url on remote server
method: "POST", // Type of request you want to issue to remote server
params: { 
    action: "getFriendsList" // Parameters you are sending to remote server
}
```
 
### STEP 2: Forward request to remote server
 
This step is a bit tricky. 
 
The client request arrives to your server-side code. First, you need to parse and validate request data.
 
To repeat once more ... you'll need the following data on server:
 
```php
$restUrl = $request->request->get('restUrl'); // Your target url on remote server
$method = $request->request->get('method'); // Type of request you want to issue to remote server
$params = $request->request->get('params'); // Parameters you are sending to remote server
$contentType = $request->request->get('contentType'); // Content-type
```
 
Create curl request and set parameters that you've got from the client.

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
 
I strongly suggest reading the PHP manual to get familiar with the curl configuration: http://php.net/manual/en/function.curl-setopt.php
 
You have enough data to send a request to the remote server. After executing the curl handle, the response from the remote server is stored in $response variable.
 
```php
$response = curl_exec($ch);
curl_close($ch);
```
 
You have a basic setup. Now we are going to add cookie support to our ajax proxy.
 
Why do we need cookies? 
 
Recently I needed to integrate Symfony 2 application with WordPress portal. Authentication is done on the Wordpress side, and Symfony application is using WordPress cookies to authorize users for accessing protected features.
 
In production, both applications share the same domain, but in the test environment, there is a lot of mess (different domains, ports, etc).
 
Where is the problem? 
 
An application written in Symfony 2 needs to consume some protected backend functionalities on the Wordpress side. There is a lot of personalized data that is fetched via ajax calls, and this is the reason why we need cookies.
 
### How to get cookies from Symfony2 request and set multiple cookies to Curl?
 
 We need to extract multiple cookies from Symfony 2 request objects and set them to curl handle.
 
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
 
### How to get cookies Curl response and set multiple cookies to Symfony response?
 
Remember when we configured curl with CURLOPT_HEADER=true? Now we are going to parse cookies from curl Http response.
 
```php
// Get header and response data from curl response
list($headers, $response) = explode("\r\n\r\n",$response,2);
// We are using regex to parse cookies from curl response
preg_match_all('/Set-Cookie: (.*)\b/', $headers, $cookies);
// Store cookies
$cookies = $cookies[1];
```

Raw cookies are parsed and stored in cookie array. Then each cookie needs to be converted to Symfony Cookie object and injected to Symfony response headers.
 
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
 
**FINAL NOTICE:**
 
Close and store session data before initializing curl handle or you can run into a deadlock.

 
```php
session_write_close();
$ch = curl_init();
```
 
What could happen if you forgot to close the session?
 
Here is one scenario:

PHP script S1 creates/sends a curl POST request to script S2 on the same Apache server/PHP Environment. If pages S1 and S2 share the same session, script S2 will not start executing until the end of script S1 lifetime. But script S1 is waiting on a response from script S2. This is the point where deadlock happens. The default session handler locks the session file for the duration of the page request.

 <a id="source-code" name="source-code">Complete source code listing for ajax proxy</a>
 
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
http://developer.yahoo.com/javascript/howto-proxy.html
 
[Follow me on Twitter](https://twitter.com/#!/svlada)
