When creating a command-and-control infrastructure, it is common for the callbacks to not communicate directly to the attacker’s C2 server. Many times, they will go through a compromised webpage, or a fake site used as a redirector. A redirector is basically a server that will take requests and forward them to another address, such as the real malicious server. This is to hide the underlying attacker address if the C2 traffic is ever discovered. 

Have you ever analyzed a web address that was flagged as malicious, only to see a seemingly benign or a 404-error page? This may indicate that the page is not malicious or no longer existing, but it could also indicate the page is being used as a redirector/proxy. 

Apache, Nginx, and other web servers have the ability to proxy/redirect traffic when desired conditions are met. This is useful from an attacking perspective as it can add an extra layer of obfuscation to an analyst observing the traffic. 

To demonstrate this, we set up a public facing Apache server at domain “*pc-tech.pro*” as a redirector and configured the [.htaccess](https://ithemes.com/what-is-the-htaccess-file/) file to proxy or redirect traffic based on different “rules”. This can be done through Apache’s [mod_rewrite](https://httpd.apache.org/docs/2.4/mod/mod_rewrite.htmlhttps://httpd.apache.org/docs/2.4/mod/mod_rewrite.html). 

![image-center](/images/C2-Redirection-for-Offensive-Operations/htaccess.png){: .align-center}

What the above .htaccess file does, is that every web request to our domain will be redirected or proxied based on the RewriteConditions. The logic of this file is as follows:

**IF:**

1) If the requested URL is any of:
* d/ref=ab_sb_ns_1/7888-0262949/field-keywords=ads
* D15/afa/amzn.us.sr.aps
* about-us
* contact-us


**AND:**

2) The user-agent string is:
* "Mozilla/5.0 (Windows NT 6.1; WOW64; Trident/7.0; rv:11.0) like Gecko"


**THEN:**

3) Proxy traffic to http://13.59.197.154 (our Cobalt Strike server) at the requested URL


**ELSE:**

4) Return a 302 (Found) response and redirect to *https://computertechpro.net* (some other legitimate site) at the requested URL

To demonstrate how these rules work, we added a plugin to a Firefox web browser to change the user-agent in the requests. We then hosted a file on our Cobalt Strike attacker server to show what conditions need to be met to observe it. 

<ins>**Making a request with the incorrect user-agent, but correct URL:**</ins>

Request: *http://pc-tech.pro/contact-us*

![image-center](/images/C2-Redirection-for-Offensive-Operations/example1.png){: .align-center}

Request: *http://pc-tech.pro/d/ref=ab_sb_ns_1/7888-0262949/field-keywords=ads*

![image-center](/images/C2-Redirection-for-Offensive-Operations/example2.png){: .align-center}

<ins>**Making a request with the correct user-agent and correct URL:**</ins>

Request: *http://pc-tech.pro/contact-us*

![image-center](/images/C2-Redirection-for-Offensive-Operations/example3.png){: .align-center}


<ins>**Making a request with any user-agent and incorrect URL:**</ins>

Request: *http://pc-tech.pro/test*

![image-center](/images/C2-Redirection-for-Offensive-Operations/example4.png){: .align-center}

# Modifying for C2 traffic

Within Cobalt Strike’s malleable C2 [framework](https://www.cobaltstrike.com/help-malleable-c2), fields such as the user-agent and callback URLs can be modified based on the infrastructure needs. For this specific configuration, the malleable C2 user-agent was set to match that of our Apache mod_rewrite rules, as well as the URLs used for GET and POST requests. 

User-Agent

![image-center](/images/C2-Redirection-for-Offensive-Operations/user-agent.png){: .align-center}

GET Requests

![image-center](/images/C2-Redirection-for-Offensive-Operations/get-request.png){: .align-center}

POST Requests

![image-center](/images/C2-Redirection-for-Offensive-Operations/post-request.png){: .align-center}

The below Wireshark capture is from the Cobalt Strike payload being executed. The user-agent and the requested URL match that of the Apache webserver configuration, so the server responds with a 200 (OK) code and our traffic is proxied to our Cobalt Strike C2 server. The resulting beacon can be seen below. 

*\*pc-tech.pro is located at IP 3.16.149.234\**

![image-center](/images/C2-Redirection-for-Offensive-Operations/wireshark.png){: .align-center}

The source IP address can be seen as 3.16.149.234, which is our Apache server.

![image-center](/images/C2-Redirection-for-Offensive-Operations/cobalt-strike.png){: .align-center}

The following Wireshark capture is that of a browser requesting the same URL, but as the user-agent string is not correct, the Apache webserver returns a 302-redirect response, and forwards the session to "*computertechpro.net*". As this page does not exist, a 404-error page is returned. 

![image-center](/images/C2-Redirection-for-Offensive-Operations/wireshark2.png){: .align-center}

Below is a diagram which illustrates the above concept on a high-level.

![image-center](/images/C2-Redirection-for-Offensive-Operations/drawio.png){: .align-center}

This example is meant to be an introduction to the concept of redirection to hide C2 infrastructure. These tactics can become much more complex based on how complex the rule sets on the redirectors are.  

These tactics can also be used in the instance of compromised domains. An attacker could compromise a legitimate website and modify the server configuration files to redirect certain users to malicious addresses. If this is a trusted site, it may not return as malicious on threat feeds or other intelligence. 

