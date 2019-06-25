---
title: Proxy Ajax requests, Curl and Symfony 2
description: This is introductory RequireJS tutorial in RequireJs series.
date: 2013-12-23
tags:
  - symfony
  - curl
layout: layouts/post.njk
permalink: "require-js-dependency-management-part1/index.html"
---
In this article I will explain, how to make cross-domain requests through the proxy using Curl and Symfony 2.
 
Jump to the [#source-code](#source-code) and skip following errata on how to proxy Ajax requests.
 
 Usual scenario looks like this:
 
 1. Client send ajax request to server
 2. Your server forwards request to external/remote server
 3. Waiting on response from remote server
 4. Parse and process response from remote server
 5. Send response back to client
 
First we shall check if client request is XmlHttpRequest using Symfony 2 built in method
 
```
 $request->isXmlHttpRequest()
```
 
### STEP 1: Client code
 
Client code is quite simple. 
 
You need to craft Ajax request in following way.
 
1. Specify rest url on your server for handling cross-domain ajax requests.`
 url: "{{ path('_ajaxProxy') }}"`
2. Wrap request data
 
We need some data in order to create curl request on server-side.

For this example you need to send object with following properties to your server.
 
```
     restUrl: "external-api-url", // Your target url on remote server
     method: "POST", // Type of request you want to issue to remote server
     params: { 
         action: "getFriendsList" // Parameters you are sending to remote server
     }
```
 
### STEP 2: Forward request to remote server
 
This step is bit tricky. 
 
Client request is arrived to your server-side code. First you need to parse and validate request data.
 
To repeat once more ... you'll need the following data on server:
 
```
<?php
 $restUrl = $request->request->get('restUrl'); // Your target url on remote server
 $method = $request->request->get('method'); // Type of request you want to issue to remote server
 $params = $request->request->get('params'); // Parameters you are sending to remote server
 $contentType = $request->request->get('contentType'); // Content-type
 ?>
```
 
You need to create curl request and set parameters you have recieved from client.

```
 <?php
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
 ?>
```
 
I strongly suggest to read php manual in order to get familiar with the curl configuration: http://php.net/manual/en/function.curl-setopt.php
 
You have enough data to send request to remote server. After executing curl handle, response from remote server is stored in $response variable.
 
```
<?php
     $response = curl_exec($ch);
     curl_close($ch);
 ?>
```
 
You have basic setup. Now we are going to add cookie support to our ajax proxy.
 
Why do we need cookies? 
 
Recently I needed to integrate symfony 2 application with wordpress portal. Authentication is done on Wordpress side, and symfony application is using wordpress cookies to authorize users for accessing protected features.
 
In production both applications share same domain, but in test enviorment there is a lot of mess (different domains, ports etc).
 
Where is problem? 
 
Application written in Symfony 2 need to consume some protected backend functionalities on Wordpress side. There is a lot personalized data that is fetched via ajax calls, and this is reason why we need cookies.
 
### How to get cookies from Symfony2 request and set mutplie cookies to Curl?
 
 We need to extract multiple cookies from Symfony 2 request object and set them to curl handle.
 
```
<?php
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
?>
```
 
### How to get cookies Curl response and set multiple cookies to Symfony response?
 
Remmember when we configured curl with CURLOPT_HEADER=true? Now we are going to parse cookies from curl http response.
 
```
<?php    
     // Get header and response data from curl response
     list($headers, $response) = explode("\r\n\r\n",$response,2);
     // We are using regex to parse cookies from curl response
     preg_match_all('/Set-Cookie: (.*)\b/', $headers, $cookies);
     // Store cookies
     $cookies = $cookies[1];
?>
```
 
Raw cookies are parsed and stored in cookie array. Then each cookie need to be converted to Symfony Cookie object and injected to Symfony response headers. 
 
```
<?php
     foreach($cookies as $rawCookie) {
         $cookie = \Symfony\Component\BrowserKit\Cookie::fromString($rawCookie);
         $value = $cookie->getValue();
         if (!empty($value)) {
             $value = str_replace(' ', '+', $value);
         }
         $customCookie = new Cookie($cookie->getName(), $value, $cookie->getExpiresTime()==null?0:$cookie->getExpiresTime(), $cookie->getPath());
         $response->headers->setCookie($customCookie);
     }
 ?>
```
 
**FINAL NOTICE:**
 
 Close and store session data before initializing curl handle or you can run into deadlock.
 
```
<?php
     session_write_close();
     $ch = curl_init();
?>
```
 
What could happen if you forgot to close session?
 
Here is one scenario:

PHP script S1 create/sends curl POST request to script S2 on same Apache server/PHP Enviorment. If page S1 and S2 share same session, script S2 will not start executing until end of script S1 lifetime. But script S1 is waiting on response from script S2. This is point where deadlock happen. The default session handler locks the session file for the duration of the page request.

 <a id="source-code" name="source-code">Complete source code listing for ajax proxy</a>
 
 **Client sample code**
```
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

```
<?php
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