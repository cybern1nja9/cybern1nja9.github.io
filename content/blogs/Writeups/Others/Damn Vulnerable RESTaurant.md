---
author: [""]
series: ["API Hacking"]
title: "Damn-Vulnerable-RESTaurant-API-Game"
description: Damn Vulnerable Restaurant is an intentionally vulnerable Web API game for learning and training purposes dedicated to developers, ethical hackers and security engineers covering most critical categories of OWASP TOP 10 for API.
summary: Writeup covering all the vulnerabilities and exploitations that are there in all six levels.
ShowToc: true
TocOpen: false
date:   2025-09-20
tags:   [API,API Hacking,OWASP Top 10 API]
boxinfo:
 image: https://raw.githubusercontent.com/GangGreenTemperTatum/Damn-Vulnerable-RESTaurant-API-Game/main/app/static/img/mad-chef-circle-text.png
---

## Lab Setup 

1. Install [Docker](https://www.docker.com/get-started/) and [Docker Compose V2](https://docs.docker.com/compose/install/).
2. For Windows, (make sure we've installed "git" for windows)
Open CMD or PowerShell as administrator
```
git clone https://github.com/theowni/Damn-Vulnerable-RESTaurant-API-Game.git
cd Damn-Vulnerable-RESTaurant-API-Game
docker compose up -d
```
3. For Linux
```
git clone https://github.com/theowni/Damn-Vulnerable-RESTaurant-API-Game.git
cd Damn-Vulnerable-RESTaurant-API-Game
./start_app.sh
```


![Test](/img/posts/20250811190901.png)
4. Now, visit http://127.0.0.1:8091/docs or http://<your_ip>:8091/docs
![Test](/img/posts/20250811191214.png)

![Test](/img/posts/20250811191048.png)

**Setup Done!**

------------------

## Tool setup

**Option 1:**
You can send request via the OpenAPI (Swagger) found at http://127.0.0.1:8091/docs  to Burp Suite via proxy

**Option 2:**
But, I'll use a Burp extension called https://github.com/nerdygenii/postman-burp-importer as I wanted to try this tool out for my workflow.

How to set up…

1. Download the latest JAR from [Releases](https://github.com/nerdygenii/postman-burp-importer/releases/latest)
2. In Burp Suite: `Extensions` → `Add` → Select the JAR file
3. The “Postman Importer” tab will appear
4. Getting the Postman Collection for the "postman-burp-importer" ext (since, we've  **OpenAPI Specification** formerly Swagger .JSON file and **not** the Postman collection which is required for the extension)

Download the API doc from http://127.0.0.1:8091/openapi.json >> Open Postman >> Import >> Files >>Select openapi.json >> Open API 3.1 with a Post collection >> Import

![Test](/img/posts/20250812205710.png)

Next, export the post collection...

![Test](/img/posts/20250812210617.png)
![Test](/img/posts/20250812210417.png)
5. Open **Burp Suite** >> go to  **Extension** >> **Postman Importer** >>  **Browse** >> **select** <**your_exported-postman_collection.json**> >> **Import Collection.**

![Test](/img/posts/20250812222724.png)

All the API requests will get imported in the requester tabs

![Test](/img/posts/20250812222852.png)


Now, It's time for some API Hacking!

## Level 0 — Information disclosure
![Test](/img/posts/20250909192232.png)

Send a **GET** request to **/healthcheck**

You will find the info disclosure in response — "**x-powered-by: Python 3.10, FastAPI ^0.103.0**", which will be useful to plan and carry out further attack.

![Test](/img/posts/20250909192749.png)
	

## Level 1 — Unrestricted Menu Item Deletion or (BFLA)**

![Test](/img/posts/20250909193350.png)

**Attack -**

Get the Auth Token for our attacker's account (**userA**) via POST /token endpoint
![Test](/img/posts/20250914210016.png)


Verifying it...

![Test](/img/posts/20250914210120.png)

	
Send a request to GET /menu (to view list of menus).

We'll try to **delete** ID: 17
![Test](/img/posts/20250914210210.png)

Now, send a DELETE request to /menu/{id}

![Test](/img/posts/20250914210400.png)

Checking, if the id: 17 got successfully deleted!? Yes, it is -
![Test](/img/posts/20250914210539.png)
*It's a BFLA finding, since we as an attacker has the **role: Customer** (low privileged account) and we were able to perform **unauthorized** + **high-level privileged function.***


## Level 2** — **BOLA (Broken Object Level Authorization)**

We'll perform "A-B" testing...so create two accounts.

One will act as an attacker's account and the other one will be the victim's account. 

Attacker - UserA
![Test](/img/posts/20250916215909.png)

Victim - UserB
![Test](/img/posts/20250916215848.png)

The PUT /profile allows user to edit their profile information

Profile info updated for userA-
![Test](/img/posts/20250916220325.png)



*Attack — 

Now, let's change the profile info of userB. 

This can be achieved by changing the username value to userB (victims account) but using the auth token of userA (attacker)


Original request

![Test](/img/posts/20250916220748.png)

Tampered request

![Test](/img/posts/20250916220541.png)
Profile info updated for userB by userA!!

**To summarize this BOLA vulnerability -**

- We are authenticated as **userA**.
    
- The API endpoint `/profile` is properly secured, meaning only authenticated users can access it. This prevents a “Broken Authentication” vulnerability, since unauthenticated users cannot access the function.
    
- However, the authorization check is flawed. Instead of verifying that the `username` in the request body matches the username associated with the authorization token, the API simply trusts the value provided in the request body.
    
- By changing the `username` from **userA** to **userB**, we were successfully able to modify an object (**userB's profile**) that we are not authorized to interact with.


 We can also send the PUT /profile request to Intruder targeting "username" parameter for enumerating and attacking other user account using common username wordlist
![Test](/img/posts/20250831181147.png)

![Test](/img/posts/20250831183013.png)

We see that we found one valid existing user called “**chef**”, which seems interesting (as it might have some elevated privileges.
![Test](/img/posts/20250831183155.png)

## Level 3 - Privilege Escalation** (via "**BOPLA**"/**Broken Object Property Level Authorization**)

![Test](/img/posts/20250916222456.png)

The API endpoint PATCH /profile allows user to update their profile info.

As per API doc — The request does not contain “role” value. We can try to add the “role” property in the request and see if we can see a successful response.

![Test](/img/posts/20250831190640.png)


![Test](/img/posts/20250831185147.png)

Tampered Request -

![Test](/img/posts/20250831190308.png)

We see that our role got changed to “Chef” as we included unintended property "**role**" in our request.
![Test](/img/posts/20250831191215.png)


## Level 4 - Server Side Request Forgery**

In the API endpoint PUT /menu, you'll see an object "image_URL". 
So, let's test this object by inserting an external URL.

![Test](/img/posts/20250916224017.png)

I injected a custom created URL from webhook.site, let's see what happens!?

![Test](/img/posts/20250916224106.png)
![Test](/img/posts/20250916224125.png)

Upon sending the request, we get a base64 encoded response.
![Test](/img/posts/20250916224250.png)


After base64 decoding, we get our content that we created. This means the api is fetching the content from the URL and displays it in base64 encoded format.
![Test](/img/posts/20250916224331.png)
This means that SSRF might be possible on this endpoint. Now, let exploit it further to make it impactful...

After, reviewing the code under - "**/Damn-Vulnerable-Restaurant-API-Game/app/apis/admin/services**” - it was found that there were two API routes -
one of which was -

- **reset_chef_password_service.py** - (responsible for generating and updating a new passwd for "admin" aka "chef")

![Test](/img/posts/20250920153705.png)

After, replacing the previous URL with an internal URL i.e., **http://localhost:8091/admin/reset-chef-password**

![Test](/img/posts/20250920153349.png)

![Test](/img/posts/20250920153841.png)
That's how, we got the admin's password. 
Now, we'll authenticate with admin cred to get the access token.

![Test](/img/posts/20250920182149.png)


Verifying it!
![Test](/img/posts/20250920182253.png)


## Level 5 — RCE**

![Test](/img/posts/20250920182652.png)

Only user with role “Chef” is authorized to send the request to the endpoint 
**GET /admin/stats/disk?parameters=** 

![Test](/img/posts/20250920184833.png)

Hence, we will use the admin's token that we have with us...(got from previous level - SSRF)

![Test](/img/posts/20250920185240.png)

The above output looks like from the Linux command **df**

**Attack -**
Let's see if we can perform command injection -

I tested out the following —> **; whoami**  (*url-encoded*)

![Test](/img/posts/20250920185653.png)
In the output, we see the **whoami** command got executed successfully.

Now, let's try to get a reverse shell...

In order to inject right set of payload - We already have a useful piece of information that the backend language being used is **Python3** as shown in the below highlight screenshot.

![Test](/img/posts/20250920191517.png)

So, we will get the payload from https://www.revshells.com/

![Test](/img/posts/20250920203137.png)

and will inject the payload as the value of "**parameter=; " 

This request will look something like this...

![Test](/img/posts/20250920203558.png)

Turning the net cat listener **ON**...

![Test](/img/posts/20250920204327.png)

Now, send the request!

We successfully got the reverse shell.
![Test](/img/posts/20250920204620.png)

![Test](/img/posts/20250920204910.png)

Now, let's try to get the root access.

![Test](/img/posts/20250920204935.png)

The user `app` can run the `/usr/bin/find` command as root (or any user) without providing a password.

This means `find` can execute arbitrary commands using the `-exec` option, an attacker, or user could escalate privileges to root by running a command like:

```
sudo find / -exec /bin/sh \; -quit
```

![Test](/img/posts/20250920205743.png)

Voilà! We got root access.

![Test](/img/posts/20250920205911.png)


<b> Note </b> - I havn't covered the fixing the vulnerabilites part and have only covered the attacking part.

-----------------------------------------

Thank you for reading😄  
धन्यवाद पढ़ने के लिए!  
अनुगृहीतोऽस्मि पठितवान्  
感谢您的阅读   
¡Gracias por leer!  
Danke fürs Lesen!  
読んでくれてありがとうございます!  
شكراً على القراءة  
Благодарю за чтение!