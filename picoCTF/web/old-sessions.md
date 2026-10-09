🚩 picoCTF Writeup: Old Sessions

Category: Web Exploitation



Difficulty: Easy



Platform: picoCTF 2026



Vulnerability: Improper Session Management / Broken Authentication / Session Leakage



Tools Used: Web Developer Tools (Inspect Element / Application Tab), cURL, Burp Suite



1\. Challenge Overview

Description:

The web developer mentions that the site is still under development. He hates constantly logging into sites, so he designed it such that "once you log in, you never have to log out again!"



Find the flag on the target server.



2\. Reconnaissance \& Initial Enumeration

Launch the instance and navigate to the application URL in a web browser.



Register a temporary test account (e.g., username: dev, password: dev123) and log in.



Inspect the homepage HTML source code and comment sections. Inside a public comment section, a user message stands out:



"Hey I found a strange page at /sessions."



Navigate directly to the discovered endpoint: http://<TARGET\_HOST>:<PORT>/sessions



3\. Vulnerability Discovery

Navigating to /sessions exposes a live dump of active user session tokens stored by the application:



Plaintext

1\) session:mL\_C8lzVGrqTt1qiyIqiRnH0YyEPDqQMGEB-a\_vrIsg, {'\_permanent': True, 'key': 'admin'}

2\) session:PzkDYsNCGgrNQYyrVObF2UvaQvMp\_RU-dAfk4rPEFyM, {'\_permanent': True, 'key': 'dev'}

Flaws Identified

Sensitive Data Exposure: Active session keys are publicly listed without authentication checks.



Persistent Sessions / No Expiration: Sessions do not invalidate upon logout or idle timeout.



Cookie-Based Authentication Trust: Session tokens directly map to privileges without secondary validation.



4\. Exploitation Walkthrough

Method 1: Using Browser Developer Tools

Open Browser Developer Tools (F12 or Ctrl + Shift + I).



Go to the Application tab (or Storage tab in Firefox).



Expand Cookies under the left sidebar and select the challenge domain.



Locate the session cookie key.



Double-click the cookie Value field and replace your test cookie with the stolen admin session token from /sessions:



Plaintext

mL\_C8lzVGrqTt1qiyIqiRnH0YyEPDqQMGEB-a\_vrIsg

Refresh the homepage (F5).



Method 2: Using cURL Command Line

Bash

\# Issue a GET request using the stolen admin session cookie

curl -s -b "session=mL\_C8lzVGrqTt1qiyIqiRnH0YyEPDqQMGEB-a\_vrIsg" http://<TARGET\_HOST>:<PORT>/ | grep -i "picoCTF"

5\. Flag Capture

Upon refreshing the homepage with the hijacked admin cookie, the web interface loads the administrative panel and renders the flag:



Plaintext

\[+] Flag Obtained: picoCTF{s3ss10n\_l34k\_4nd\_f0r3v3r\_c00k13\_4b31a89c}

6\. Root Cause \& Mitigation

Root Cause: The application explicitly retains session states indefinitely without invalidation routines and exposes internal session state data via an unauthenticated endpoint (/sessions).



Remediation:



Restrict Access: Restrict or remove debug endpoints like /sessions in production environments.



Session Timeout: Enforce strict session expiration limits (idle timeouts and absolute timeouts).



Secure Cookies: Store session identifiers securely server-side and issue cookies with HttpOnly, Secure, and SameSite flags.

