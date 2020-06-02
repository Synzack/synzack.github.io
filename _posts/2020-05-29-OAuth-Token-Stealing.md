# Preface 

The techniques presented in this paper are not necessarily new and were not initially discovered by myself. There is notable research that was done by [FireEye](https://www.fireeye.com/blog/threat-research/2018/05/shining-a-light-on-oauth-abuse-with-pwnauth.html) and [MDSec](https://www.mdsec.co.uk/2019/07/introducing-the-office-365-attack-toolkit/) prior to this publication that sparked this report. The goal of this post is to build on their research, give a background of the techniques used, and present new tools for security teams to use as testing frameworks in the future. 

In addition, this paper only covers OAuth abuse from the perspective of Microsoft accounts. This technique can be implemented on any service using similar OAuth protocols or permissions. Whether it be mobile apps, social media, personal email, or professional accounts, be sure to always review the permissions requested and the request source when granting third party access. 

# OAuth 

## What is OAuth? 

OAuth is an open standard authorization protocol/framework that make it possible for applications, servers, and other unrelated services to have a way to have secure authenticated access. The protocol is designed to be able to do this without sharing any logon credentials (such as the user’s actual password). If you would like to learn more how the protocol works, check out this [article](https://www.csoonline.com/article/3216404/what-is-oauth-how-the-open-authorization-framework-works.html).

OAuth was released in 2010 and has been expanded upon in OAuth 2.0, released in 2012. It has become a widely used authentication platform used by corporations such as Amazon, Facebook, and Microsoft.  

The general operational flow for this authentication is as follows: 

1. A user needs to authenticate to a website or service outside of their organization 
2. The resource forwards the user to a second authorization website on behalf of the user, where OAuth is used to provide the user’s identity 
3. If not already authenticated, the user may be asked to provide credentials 
4. The second site confirms the user’s identity and returns an access-token to the first website 
5. The first site uses this token to authenticate to the necessary services on behalf of the user 

Below is a general diagram of the OAuth flow from [Digital Ocean](https://www.digitalocean.com/community/tutorials/an-introduction-to-oauth-2). 

![image-center](/images/OAuth-Token-Stealing/Dig-Ocean-OAuth.jpeg){: .align-center}

This is an oversimplified explanation, but you may use OAuth in your everyday workflow without even knowing. For example, let’s say you want to access your Office365 applications such as OneDrive, Office, and SharePoint. Here is the flow: 

1. You go to ‘Office.com’ and click the button to sign-in.  
2. You enter your corporate email address, and the site says, “Taking you to your organization’s sign-in page”  
3. You are forwarded to the corporate Okta (or other sign-in) page, where you input your domain credentials and two-factor authentication (if applicable)
4. Once authenticated to your organization, you are redirected back to ‘Office.com’ and have all your applications available to you 
5. Behind the scenes, an access-token was given to Office365 to authenticate you, but your domain password was never exchanged  

## Microsoft Graph API

In many modern-day environments, Microsoft Office products are typically integrated to increase productivity, collaboration, and create deliverables. Some common examples are the Microsoft Office Suite (Word, Excel, PowerPoint, etc.) for projects, Outlook for email, SharePoint for sharing company resources, and OneDrive for cloud storage.  

To further integrate these tools, Microsoft as introduced the [Microsoft Graph API](https://docs.microsoft.com/en-us/graph/use-the-api), which is a [RESTful](https://restfulapi.net/) web API that enables users and applications to access these services through their request system. In order to use these tools, an access-token needs to be granted to the application, through the same OAuth protocols we discussed above.  

These API calls can be used to perform many actions on behalf of the user including reading email, sending email, accessing files, reading user information, and much more. If there is an action you would like to perform on a Microsoft cloud product, there is likely an API call for that. For a full list of permissions that can be granted to an app, please visit the official Microsoft permission references [documentation](https://docs.microsoft.com/en-us/graph/permissions-reference). 

If you have ever seen a prompt such as the following, you have likely interreacted this service in some way, shape, or form.  

![image-center](/images/OAuth-Token-Stealing/permission-prompt.jpeg){: .align-center}

Below are graphic from the Microsoft [documentation](https://docs.microsoft.com/en-us/graph/overview) with a visualization of the Graph API.

![image-center](/images/OAuth-Token-Stealing/ms-graph1.jpeg){: .align-center}

![image-center](/images/OAuth-Token-Stealing/ms-graph2.jpeg){: .align-center}
<br>

# Room for Abuse

## What are the Security Implications?

If you are reading this, you probably know that we are not here to discuss all the great, legitimate things these tools can be used for. Like most technology, if it can be used for good, it likely can be used for bad. So, if that is the case, what are the negative security implications of this technology? 

On one hand, this method of authentication protects the user’s credential material. It also requires authorization from the user and, if configured, their organization. The issue arises when a rogue site or application requests these same access tokens and is given approval through methods such as phishing and social engineering, where security controls are not in place to block the authorization. We will be focusing on this scenario for the purposes of this paper. 

Once granted permissions, the malicious service now has an access token that can act on behalf of the user for each service it has permissions to. One of the most interesting parts is that since no credential material is exchanged, the service will still have access after a password reset by the user. This also means this method can bypass multi-factor authentication. The access is only revoked if the token expires, or the permissions are explicitly revoked by either the user or the organization’s Office365/Azure administrator. 

This technique was famously used in recent years by APT28 (most commonly known as Fancy Bear, Sofacy, and Pawn Storm). The group implemented OAuth phishing techniques within their 2016 campaigns against the German Christian Democratic Union (CDU), Turkish government, and arguably most famously (at least in the USA) the Democratic National Convention (DNC). Trend Micro has an excellent [article](https://blog.trendmicro.com/trendlabs-security-intelligence/pawn-storm-abuses-open-authentication-advanced-social-engineering-attacks/) on this topic.

Through these technologies, this Russian nation-state group was able to gain unauthorized access to emails, files, and other sensitive information.  

Below is a graphic from Trend Micro's article which illustrates the attack chain of the above campaign. 

![image-center](/images/OAuth-Token-Stealing/pawnstorm.jpeg){: .align-center}

# How do these Applications Work?

To gain an understanding of how these technologies can be abused, we will walk through how to create a proof of concept rogue application from a Red Team perspective. We will create a simple application using the Microsoft documentation.

The first step is to head over to the Azure portal at [https://portal.azure.com](https://portal.azure.com). Once you sign in with a Microsoft account, you should be presented with the home panel. To get to the application management page, we need to go to “Manage Azure Active Directory.” 

![image-center](/images/OAuth-Token-Stealing/azure1.jpeg){: .align-center}

On the next page, you should see a panel that includes “App registrations."  

![image-center](/images/OAuth-Token-Stealing/azure2.jpeg){: .align-center}

Now, go to “New Registration” 

![image-center](/images/OAuth-Token-Stealing/azure3.jpeg){: .align-center}

Give the application a name and set the supported account types to your choosing. For this demonstration, we chose accounts in any directory and personal Microsoft accounts to cover all target scenarios. We will set redirects later, so do not worry about this for now. Once complete, go ahead and click “Register”. 

![image-center](/images/OAuth-Token-Stealing/azure4.jpeg){: .align-center}

After the app is registered, we can see a quickstart guide. I found this to be a great resource in creating an application. 

![image-center](/images/OAuth-Token-Stealing/azure5.jpeg){: .align-center}

For this demonstration we will be using a web application. 

![image-center](/images/OAuth-Token-Stealing/azure6.jpeg){: .align-center}

I prefer Python for creating my tools, so we will pick their Python option for this demonstration. 

![image-center](/images/OAuth-Token-Stealing/azure7.jpeg){: .align-center}

Here we will find sample code and instructions on how to get started with a simple python-based web application. 

![image-center](/images/OAuth-Token-Stealing/azure8.jpeg){: .align-center}

# Creating an Application

Step 1) Configure your application in Azure Portal 

+ Add a reply URL: 
    + Head over to the “Authentication” tab on our app panel 

![image-center](/images/OAuth-Token-Stealing/azure9.jpeg){: .align-center}

+ Add a platform

![image-center](/images/OAuth-Token-Stealing/azure10.jpeg){: .align-center}

+ Choose Web Application

![image-center](/images/OAuth-Token-Stealing/azure11.jpeg){: .align-center}

+ Add the redirect URL (This can be any web server running your application)

![image-center](/images/OAuth-Token-Stealing/azure12.jpeg){: .align-center}

Step 2) Create a Client Secret

+ Go to the “Certificates & secrets” panel. Create a new secret

![image-center](/images/OAuth-Token-Stealing/azure13.jpeg){: .align-center}

![image-center](/images/OAuth-Token-Stealing/azure14.jpeg){: .align-center}

+ Add a name and expiration. Once confirmed, you should have secret value

![image-center](/images/OAuth-Token-Stealing/azure15.jpeg){: .align-center}


![image-center](/images/OAuth-Token-Stealing/azure16.jpeg){: .align-center}

We now have everything we need to start creating our Python Application. Go ahead and download the code sample from the quickstart guide page and install requirements if you haven't already.  

There are two main programs we will be concerned with, ‘app.py’ and ‘app_config.py’. These are the backbone for our application.  

To configure for use, we need to edit the app_config.py file with our client secret and client ID. These can be found on our application panel.  

![image-center](/images/OAuth-Token-Stealing/config1.jpeg){: .align-center}

Notice the REDIRECT_PATH variable is the redirect URL we configured in our setup, if this were setup to a different path, we would need to change it here.  

SCOPE is used to define the permissions needed. For the first demonstration, “*User.ReadBasic.All*’ will work with a work or school account. If you are using a personal account, add '*User.Read*.’  

SCOPE will also need to be modified accordingly when adding more API calls as each requires a different permission set. 

Now, with the configuration file set up, our application should be ready to test. *Run the app.py* program. This will create a web server using [Flask](https://flask.palletsprojects.com/en/1.1.x/). By default, it runs on localhost:5000, which is our redirect URL in the configuration.

![image-center](/images/OAuth-Token-Stealing/app1.jpeg){: .align-center}

In a browser, the application index page looks like the following:

![image-center](/images/OAuth-Token-Stealing/app2.jpeg){: .align-center}

When we click on “Sign-in” we are redirected to Microsoft to authenticate with OAuth.  

![image-center](/images/OAuth-Token-Stealing/app3.jpeg){: .align-center}

Once authenticated, Microsoft will ask the user if they want to allow this application to access their info, based on the permissions outlined in our SCOPE. As we only have the user read permissions, it is the only permission requested in the prompt. 

![image-center](/images/OAuth-Token-Stealing/app4.jpeg){: .align-center}

Once accepted, we can see a request to our token page and the user is authenticated with the application. Behind the scenes, an access-token was returned to our web application. 

![image-center](/images/OAuth-Token-Stealing/app5.jpeg){: .align-center}

Microsoft has included a basic call to their Graph API in the demo, which we can see with "Call Microsoft Graph API” hyperlink.  

When we go to this page, we can see a JSON response to the API call, displaying the user information. 

![image-center](/images/OAuth-Token-Stealing/app6.jpeg){: .align-center}

We have successfully called the Microsoft Graph API! 

# Breaking Down the Application API Calls 

To make sure we know what is going on under the hood, let’s take a high-level look at the sample code that calls the API. I encourage you to review and understand the rest of the code, but we will not go over the whole program in this paper.  

In the *app.py* code, we can see the API call in the */graphcall* Flask route. This code contains the actual API call used on the page. 

![image-center](/images/OAuth-Token-Stealing/api1.jpeg){: .align-center}

Here is the API call code:  

*graph_data = requests.get(app_config.ENDPOINT, headers={'Authorization': 'Bearer ' + token['access_token']},).json()*

This is the heart of the application’s API. Let’s add some print statements to understand the actual request.  

![image-center](/images/OAuth-Token-Stealing/api2.jpeg){: .align-center}

When we restart the web server and call the Graph API, we get the following output. 

![image-center](/images/OAuth-Token-Stealing/api3.jpeg){: .align-center}

Breaking it down, the function is making an HTTP GET request to: 

*‘https://graph.microsoft.com/v1.0/users’ using the header: ”’Authorization': 'Bearer ' + UserAccesstoken”*

According to the Microsoft Graph API documentation for "[*Get a user*](https://docs.microsoft.com/en-us/graph/api/user-get?view=graph-rest-1.0&tabs=http)", this is the exact API call you need to make to get the user information.  

The token you see is what is passed to the application from Microsoft when the user authenticates with OAuth, along with some other parameters. The most important being the access-token, and the refresh-token. By default, the access-token has a lifespan of one hour and needs to be refreshed by calling the API with the refresh-token. By default, the refresh token is valid for 14 days. 

With this token, the application has full access to any resources the user allowed in the permission prompt. With this token, we can create our own requests to the API and access resources such as emails, files, and shared sites. 

For documentation on the different types of functionality and parameters, please visit the Microsoft Graph REST API v1.0 [Reference](https://docs.microsoft.com/en-us/graph/api/overview?%5C=&view=graph-rest-1.0).

# Tools for Red Teams 

## PwnAuth

For red teamers and security researchers, there are already existing frameworks that can be used to test this activity. The first tool goes by the name ‘[*PwnAuth*](https://github.com/fireeye/PwnAuth)’ and was published in May of 2018 by FireEye. This tool uses a combination of Nginx, Django, and Docker to create an interactive web console for security researchers to test the techniques mentioned above.  

This tool currently has the following functionality: 

+ Reading mail messages 
+ Searching the user's mailbox 
+ Reading the user's contacts 
+ Downloading messages and attachments 
+ Searching OneDrive and downloading files 
+ Sending messages on behalf of the user 

Below is a quick snippet from their [website](https://www.fireeye.com/blog/threat-research/2018/05/shining-a-light-on-oauth-abuse-with-pwnauth.html) of their web GUI: 

![image-center](/images/OAuth-Token-Stealing/pwnauth.jpeg){: .align-center}

## Office 365 Attack Toolkit

Another tool worth mentioning is the [*Office 365 Attack Toolkit*](https://github.com/mdsecactivebreach/o365-attack-toolkit), published by MDSec in July 2019. This tool uses a web framework written in [Go](https://golang.org/) with an SQLite database backend to create a similar web interface for security researchers to test their environments against OAuth token stealing with malicious applications. 

This tool currently has the following functionality: 

+ Extraction of keyworded e-mails from Outlook. 
+ Creation of Outlook Rules. 
+ Extraction of files from OneDrive/Sharepoint. 
+ Injection of macros on Word documents. 

Below is a quick snippet from their GitHub documentation: 

![image-center](/images/OAuth-Token-Stealing/o365toolkit.jpeg){: .align-center}

## PynAuth

While the above tools are very useful and well put together, I had some learning curve when attempting to configure them properly. My tools of preference are those which I can quickly setup and takedown and can be quickly modified as needed, with little overhead.  

As part of this research, I created a Python-based framework called [*PynAuth*](https://github.com/Synzack/PynAuth) which is a tool that utilizes the same techniques as the above tools. The difference is that it is written completely in Python. It is designed to be modular, and can be run right your terminal. The tool is meant to be easy to use and easily customizable depending on which API calls you wish to implement.  

The tool’s name is a play on the *PwnAuth* tool name written by FireEye. **Please note, while this tool is functional, it may contain bugs and is currently in a beta stage* 

Current capabilities: 

+ Get user information 
+ Send email on behalf of the user 
+ Query users’ email inbox for keywords, display message, and download attachments 
+ Access users’ OneDrive folders and download files as desired 
+ Create email inbox rules 
+ List/Delete email inbox rules 
+ Refresh the users’ tokens, which allow for access up to 14 days 

Terminal View:

![image-center](/images/OAuth-Token-Stealing/pynauth.jpeg){: .align-center}

Example: Query Email

![image-center](/images/OAuth-Token-Stealing/pynauth2.jpeg){: .align-center}

![image-center](/images/OAuth-Token-Stealing/pynauth3.jpeg){: .align-center}

![image-center](/images/OAuth-Token-Stealing/pynauth4.jpeg){: .align-center}

# Defending Against OAuth Application Attacks 

While these techniques may provide a stealthy means to keep persistence in a target network, there are many measures that can be taken to prevent these attacks before they happen. Blocks for these applications will typically require proper configuration, as by default, no admin permission is needed for most resources the user already has permission to. These include things like read/write permissions to personal email and accessing both personal and corporate cloud storage resources. 

Some security measures that can be implemented include: 

+ Limiting the permissions that can be requested by third-party applications 
+ Banning third-party applications altogether  
+ Creating application white lists to include only those in use by your organization 
+ Query your organization’s users and the applications they have granted permissions to 
+ Logging user consent events within your cloud environment  

FireEye has even published a PowerShell script which can help administrators hunt for malicious applications within their cloud environments. The tool can be found on [GitHub](https://github.com/dmb2168/OAuthHunting). 

If you are an end user, be sure to always review third party permissions before granting access to your accounts, whether personal or professional. When in doubt, confirm with your administrator whether the application is legitimate or not. 

