---
author: ["Abhishekl"]
series: ["API Hacking"]
title: "Damn-Vulnerable-RESTaurant-API-Game"
description: A writeup for Damn-Vulnerable-RESTaurant-API-Game
summary: This writeup details covers all the vulnerabilities and exploitation that are there in all six levels. 
ShowToc: true
TocOpen: false
date:   2025-20-09
tags:   [API, API Hacking, OWASP Top 10 API]
boxinfo:
image: https://htb-mp-prod-public-storage.s3.eu-central-1.amazonaws.com/avatars/5837ac5e28291146a9f2a8a015540c28.png
---

#LabSetup 

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


![[Pasted image 20250811190901.png]]
4. Now, visit http://127.0.0.1:8091/docs or http://<your_ip>:8091/docs
![[Pasted image 20250811191214.png]]

![[Pasted image 20250811191048.png]]

**Setup Done!**

------------------

**-Tool setup**

**Option 1:**
You can send request via the OpenAPI (Swagger) found at http://127.0.0.1:8091/docs  to Burp Suite via proxy

**Option 2:**
But, I'll use a Burp extension called https://github.com/nerdygenii/postman-burp-importer as I wanted to try this tool out for my workflow.

- How to set up…

1. Download the latest JAR from [Releases](https://github.com/nerdygenii/postman-burp-importer/releases/latest)
2. In Burp Suite: `Extensions` → `Add` → Select the JAR file
3. The “Postman Importer” tab will appear
4. Getting the Postman Collection for the "postman-burp-importer" ext (since, we've  **OpenAPI Specification** formerly Swagger .JSON file and **not** the Postman collection which is required for the extension)

Download the API doc from http://127.0.0.1:8091/openapi.json >> Open Postman >> Import >> Files >>Select openapi.json >> Open API 3.1 with a Post collection >> Import

![[Pasted image 20250812205710.png]]

Next, export the post collection...
![[Pasted image 20250812205903.png]]
![[Pasted image 20250812210617.png]]
![[Pasted image 20250812210417.png]]
5. Open **Burp Suite** >> go to  **Extension** >> **Postman Importer** >>  **Browse** >> **select** <**your_exported-postman_collection.json**> >> **Import Collection.**

![[Pasted image 20250812222724.png]]

All the API requests will get imported in the requester tabs

![[Pasted image 20250812222852.png]]
------------------------------------------------------------------------------------------------------

Now, It's time for some API Hacking!

**Level 0** — 
![[Pasted image 20250909192232.png]]

Send a **GET** request to **/healthcheck**

You will find the info disclosure in response — "**x-powered-by: Python 3.10, FastAPI ^0.103.0**", which will be useful to plan and carry out further attack.

![[Pasted image 20250909192749.png]]
	

**Level 1 — Unrestricted Menu Item Deletion or (BFLA)**

![[Pasted image 20250909193350.png]]

**Attack -**

Get the Auth Token for our attacker's account (**userA**) via POST /token endpoint
![[Pasted image 20250914210016.png]]


Verifying it...

![[Pasted image 20250914210120.png]]

	
Send a request to GET /menu (to view list of menus).

We'll try to **delete** ID: 17
![[Pasted image 20250914210210.png]]

Now, send a DELETE request to /menu/{id}

![[Pasted image 20250914210400.png]]

Checking, if the id: 17 got successfully deleted!? Yes, it is -
![[Pasted image 20250914210539.png]]
*It's a BFLA finding, since we as an attacker has the **role: Customer** (low privileged account) and we were able to perform **unauthorized** + **high-level privileged function.***


**Level 2** — **BOLA (Broken Object Level Authorization)**

We'll perform "A-B" testing...so create two accounts.

One will act as an attacker's account and the other one will be the victim's account. 

Attacker - UserA
![[Pasted image 20250916215909.png]]

Victim - UserB
![[Pasted image 20250916215848.png]]

The PUT /profile allows user to edit their profile information

Profile info updated for userA-
![[Pasted image 20250916220325.png]]



*Attack — 

Now, let's change the profile info of userB. 

This can be achieved by changing the username value to userB (victims account) but using the auth token of userA (attacker)


Original request

![[Pasted image 20250916220748.png]]

Tampered request

![[Pasted image 20250916220541.png]]
Profile info updated for userB by userA!!

**To summarize this BOLA vulnerability -**

- We are authenticated as **userA**.
    
- The API endpoint `/profile` is properly secured, meaning only authenticated users can access it. This prevents a “Broken Authentication” vulnerability, since unauthenticated users cannot access the function.
    
- However, the authorization check is flawed. Instead of verifying that the `username` in the request body matches the username associated with the authorization token, the API simply trusts the value provided in the request body.
    
- By changing the `username` from **userA** to **userB**, we were successfully able to modify an object (**userB's profile**) that we are not authorized to interact with.


 We can also send the PUT /profile request to Intruder targeting "username" parameter for enumerating and attacking other user account using common username wordlist
![[Pasted image 20250831181147.png]]

![[Pasted image 20250831183013.png]]

We see that we found one valid existing user called “**chef**”, which seems interesting (as it might have some elevated privileges.
![[Pasted image 20250831183155.png]]

**Level 3 - Privilege Escalation** (via "**BOPLA**"/**Broken Object Property Level Authorization**)

![[Pasted image 20250916222456.png]]

The API endpoint PATCH /profile allows user to update their profile info.

As per API doc — The request does not contain “role” value. We can try to add the “role” property in the request and see if we can see a successful response.

![[Pasted image 20250831190640.png]]


![[Pasted image 20250831185147.png]]

Tampered Request -

![[Pasted image 20250831190308.png]]

We see that our role got changed to “Chef” as we included unintended property "**role**" in our request.
![[Pasted image 20250831191215.png]]


**Level 4 - Server Side Request Forgery**

In the API endpoint PUT /menu, you'll see an object "image_URL". 
So, let's test this object by inserting an external URL.

![[Pasted image 20250916224017.png]]

I injected a custom created URL from webhook.site, let's see what happens!?

![[Pasted image 20250916224106.png]]
![[Pasted image 20250916224125.png]]

Upon sending the request, we get a base64 encoded response.
![[Pasted image 20250916224250.png]]


After base64 decoding, we get our content that we created. This means the api is fetching the content from the URL and displays it in base64 encoded format.
![[Pasted image 20250916224331.png]]
This means that SSRF might be possible on this endpoint. Now, let exploit it further to make it impactful...

After, reviewing the code under - "**/Damn-Vulnerable-Restaurant-API-Game/app/apis/admin/services**” - it was found that there were two API routes -
one of which was -

- **reset_chef_password_service.py** - (responsible for generating and updating a new passwd for "admin" aka "chef")

![[Pasted image 20250920153705.png]]

After, replacing the previous URL with an internal URL i.e., **http://localhost:8091/admin/reset-chef-password**

![[Pasted image 20250920153349.png]]

![[Pasted image 20250920153841.png]]
That's how, we got the admin's password. 
Now, we'll authenticate with admin cred to get the access token.

![[Pasted image 20250920182149.png]]


Verifying it!
![[Pasted image 20250920182253.png]]


**Level 5 — RCE**

![[Pasted image 20250920182652.png]]

Only user with role “Chef” is authorized to send the request to the endpoint 
**GET /admin/stats/disk?parameters=** 

![[Pasted image 20250920184833.png]]

Hence, we will use the admin's token that we have with us...(got from previous level - SSRF)

![[Pasted image 20250920185240.png]]

The above output looks like from the Linux command **df**

**Attack -**
Let's see if we can perform command injection -

I tested out the following —> **; whoami**  (*url-encoded*)

![[Pasted image 20250920185653.png]]
In the output, we see the **whoami** command got executed successfully.

Now, let's try to get a reverse shell...

In order to inject right set of payload - We already have a useful piece of information that the backend language being used is **Python3** as shown in the below highlight screenshot.

![[Pasted image 20250920191517.png]]

So, we will get the payload from https://www.revshells.com/

![[Pasted image 20250920203137.png]]

and will inject the payload as the value of "**parameter=; " 

This request will look something like this...

![[Pasted image 20250920203558.png]]

Turning the net cat listener **ON**...

![[Pasted image 20250920204327.png]]

Now, send the request!

We successfully got the reverse shell.
![[Pasted image 20250920204620.png]]

![[Pasted image 20250920204910.png]]

Now, let's try to get the root access.

![[Pasted image 20250920204935.png]]

The user `app` can run the `/usr/bin/find` command as root (or any user) without providing a password.

This means `find` can execute arbitrary commands using the `-exec` option, an attacker, or user could escalate privileges to root by running a command like:

```
sudo find / -exec /bin/sh \; -quit
```

![[Pasted image 20250920205743.png]]

Voilà! We got root access.

![[Pasted image 20250920205911.png]]
