This page analyses the impact of Chromium's navigator.sendBeacon bug 490015 against CORS CSRF protections
and Firefox's navigator.sendBeacon bug 1080987.

If you are aware of the navigator.sendBeacon then you can skip to section [Chromium/Chrome]
# Introduction :


## navigator.sendBeacon:

The navigator.sendBeacon() method can be used to asynchronously transfer a small amount of data over HTTP to a web server.

### Syntax

navigator.sendBeacon(url [, data]);

### Parameters

url:

    The url parameter indicates the resolved URL to which the data is to be transmitted.

data: Optional

    The data parameter is a ArrayBufferView, Blob, DOMString, or FormData object containing the data to be transmitted. 

Return values:

The sendBeacon() method returns true if the user agent is able to successfully queue the data for transfer, Otherwise it returns false.


The integreting thing about the sendBeacon here is the data parameter accepts a Blob object.

## Bolb:

A Blob object represents a file-like object of immutable, raw data. Blobs represent data that isn't necessarily in a JavaScript-native format.
To construct a Blob from other non-blob objects and data, use the Blob() constructor. 


### Blob Constructor

Blob(blobParts[, options]):

    Returns a newly created Blob object whose content consists of the concatenation of the array of values given in parameter. 


### Properties

Blob.size: Read only

    The size, in bytes, of the data contained in the Blob object.
Blob.type: Read only

    A string indicating the MIME type of the data contained in the Blob. If the type is unknown, this string is empty. 




# Chromium/Chrome:

Chromium/Chorme 57 allowed to set arbitrarty Blob.type. So, By setting the Blob.type attacker can control only the Content-Type HTTP request header.

## Links:
https://developers.google.com/web/updates/2017/06/chrome-60-deprecations#temporarily_disable_navigatorsendbeacon_for_some_blobs
https://bugs.chromium.org/p/chromium/issues/detail?id=490015

Lets test with the vulnerable version of Chromium and see what parts of HTTP request the attacker can control.

Chromium Version(Vulnerable) 57.0.2987.98 Built on 8.7, running on Debian 8.7 (64-bit)

```
navigator.sendBeacon('http://localhost/',new Blob(['any-data'], {type: 'any/content-type'}))

enjoy@debain:~$ sudo nc -l -k -p  80
POST / HTTP/1.1
Host: localhost
Connection: keep-alive
Content-Length: 8
Cache-Control: max-age=0
Origin: http://www.tata.com
User-Agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/57.0.2987.98 Safari/537.36
Content-Type: any/content-type
Accept: */*
Referer: http://www.tata.com/
Accept-Encoding: gzip, deflate, br
Accept-Language: en-US,en;q=0.8

any-data

```
On a patched Chrome Version 65.0.3325.181 (Official Build) (64-bit):

```
navigator.sendBeacon('http://192.168.0.5/',new Blob(['any-data'], {type: 'any/content-type'}))

VM103:1 Uncaught DOMException: Failed to execute 'sendBeacon' on 'Navigator': sendBeacon() with a Blob whose type is not any of the CORS-safelisted values for the Content-Type request header is disabled temporarily. See http://crbug.com/490015 for details.
    at <anonymous>:1:1
```
# Firefox:

Firefox 35 doesnot use  to send the  Origin Header and have not  treated seadBeacon requests as CORS. This is equivallent to a POST form submit.

Links: 
https://www.mozilla.org/en-US/security/advisories/mfsa2015-03/
https://bugzilla.mozilla.org/show_bug.cgi?id=1080987


Firefox ESR Debian 52.7.3 (64-bit) -Not vulnerable to this bug

```
navigator.sendBeacon('http://localhost/',new Blob(['any-data'], {type: 'any/content-type'}))

enjoy@debain:~$ sudo nc -l -k -p  80
OPTIONS / HTTP/1.1
Host: localhost
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:52.0) Gecko/20100101 Firefox/52.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate
Access-Control-Request-Method: POST
Access-Control-Request-Headers: content-type
Origin: http://php.net
DNT: 1
Connection: keep-alive

```

# Flash:

Flash player 7 and 8 use to have no restrictions on Addidng arbitrary headers. 
But that was fixed in a security patch to Flash palyer 7 around 2007.

(Copied from Medium.com)

Flash with 307 redirects can still be abused to achieve the same primitive for cross origin requests with an arbitrary content type. 


Summary (Copied from Medium.com):
 
Established best practices include verifying the Origin header, sending custom headers, and the Synchroniser Token pattern. 


# The Origin Headers role in CSRF defense:

The Origin header was proposed with the very poupose of defending against CSRF attacks.

## Links: 

Section 5 in https://seclab.stanford.edu/websec/csrf/csrf.pdf (from owasp CSRF_Prevention_Cheat_list page)

Copied from Page 8 of above pdf

Server Behavior:
To use the Origin header as a CSRF defense, sites should behave as follows:

1. All state-modifying requests, including login requests,must be sent using the POST method [6]. In particular, 
state-modifying GET requests must be blocked in order to address the forum poster threat model.

2. If the Origin header is present, the server must reject any requests whose Origin header contains an undesired value (including null).
For example, a site could reject all requests whose Origin indicated the request was initiated from another site.


# Final Thoughts:

Verifying Origin Header presense+ Domain value in Origin Header can be used to block CSRF attacks.
Synchroniser Token pattern adds additinal layer of security in case of client side security bugs of high severity.
Adding anti-csrf tokens to existing REST api can be a huge effort.A simple trade of can be to move the cookie/session token to a header.

# References:
https://medium.com/@longtermsec/chrome-just-hardened-the-navigator-beacon-api-against-cross-site-request-forgery-csrf-690239ccccf

https://developer.mozilla.org/en-US/docs/Web/API/Navigator/sendBeacon
https://developer.mozilla.org/en-US/docs/Web/API/Blob

https://www.cnet.com/products/adobe-flash-player-9/preview/ Falsh player 9 is from 2006
https://www.engadget.com/2010/06/10/adobe-flash-player-10-1-now-officially-available-for-download/ Flash player 10 from 2010

https://helpx.adobe.com/flash-player/kb/arbitrary-headers-sent-flash-player.html
https://helpx.adobe.com/flash-player/kb/actionscript-error-send-action-contains.html

[hromium/Chrome]: https://github.com/Sravan-Apps/securitynotes/tree/master/sendBeacon_CORS#chromiumchrome