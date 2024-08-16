---
title: "Finding and Fixing a Simple IDOR on the Avatao Challenge"
date: 2024-08-16 14:45:21 +0300
categories: [Web Security, web]
tags: [web, hacking, idor]
image: /assets/img/idor.jpeg
alt: "Simple IDOR finding and fixing"
---
I came across a challenge on the avatao.com site. Very simple IDOR. The site has a couple of challenges and this was my first attempt at these online challenges i never do.

The challenge addresses a brocken Access Control issue. Challenges are very similar to the portswigger academy labs.
Let's dive in.

## Walkthrough
- After accessing the link, i *started the exercise*.
![error](/assets/img/start-exercise.png)
- Click *webservice* to start the lab in a new tab
![error](/assets/img/start-webservice.png)
- Register on the portal and log in using the new credentials.
![error](/assets/img/portal.png)
- The goal is to find the nickname of a suspicious user. The *profile* endpoint shows my id as 20 (id=20) 
```text
/webservice/profile?id=20
```
- Using burp intruder, we can fuzz from 1 through 30, revealing the user id 19 details.
![error](/assets/img/attacker-profile.png)
- The next challenge was to fix this IDOR. We have the following python code, access_control.py that handles this logic. 
```code
def authorize_user(current_user: int, id_: int):
    return True
```
- The function should return `True` if the `id_` parameter matches the `current_user` parameter (the ID of the user who sent the request). Both parameters are integers.
- This can be done using a simple if statement. If the `current_user` and `id_` match, it will return `True`, hense `False`.
```code
def authorize_user(current_user: int, id_: int):
  # we add this code to perform a check to match current_user to the entered id.
    if current_user == id_:
        return True
    else:
        return False
```
- Click on `deploy` to save configuration and restart the server.
![fix-idor](/assets/img/fix-idor.png)
- Now visit the suspicious user's profile `/webservice/profile?id=19` and see the `Authorization error`.
![fixed-idor](/assets/img/idor-not-auth.png)
There. We fixed this at the basic level using a simple check.