# Recon

## Initial Reconnaissance
---

This step involves visiting the main site, signing up for an account, and perusing the site with Burpsuite open and spidering. Don't even try and find bugs. Just notice the website, the technologies in use, and most importantly just single out individual functionalities. Pay attention to the sign up flow and whatnot but don't bother looking at Burpsuite just yet.

#### Sign Up Flow

The one thing you should do with Burp is make sure your Burp History is clear and it's only logging what is in scope. Then just sign up for an account like normal. Sign out, then sign in again. All the sign up and sign in flow will be captured in Burpsuite and you can export the requests and responses into your notes for later viewing. Don't do anything else. Just note the flow and technologies in use. 

Later once you begin the actual hunting phase, you will create another account to test for authentication or authorization bugs, IDORs, etc. 

#### Functionality and Technology Identification

After you've signed back in just browse the site as if you're a regular user. Don't look at Burp, don't try and fuzz anything, and don't worry about testing anything at all. Just note (and maybe jot down) the individual functions of the site. File upload, store creation, user profile edit, add to cart, purchase, add a bank account, post a message, etc. Check out Wappalyzer and/or BuiltWith to see what powers the site. 

#### Documentation and APIs

Depending on what kind of site it is, read any API documentation, etc. Play with the API a bit if applicable. Use unique functionalities that maybe not all sites have. 

---





## Asset Discovery
---

#### Horizontal Asset Discovery

**Tools Used**:
- http://bgp.he.net
- amass intel 
- whois
- armada/masscan

**Whois**

Ping the main hostname to get an IP Address

`# whois -h whois.cymru.com <ip_address>`
`# whois -h whois.radb.net -- '-i origin <ASN>' |grep route |awk '{print $2}'

Then you can scan the CIDR ranges with armada. (cidrs.txt contains CIDR ranges separated by comma)

`# armada -p 80,443,3000,4080,4443,8080,8443 $(cat cidrs.txt)`

**Amass Intel**

`# amass intel -d <target> -whois`
`# amass intel -df <file> -whois`
`# amass intel -cidr <cidr_range> -whois`
`# amass intel -asn <asn> -whois`

Only useful if all inclusive scope and any assets owned by the company are in scope

#### Subdomain Enumeration

**Tools Used**:
- amass enum

`# amass enum -d <target> -active -brute -bl blacklist.txt -nf <already_found_subdomains>.txt -ipv4`

`# amass db -names -d <target> | httpx -nc -sc -fc 404 -fc -title -location -server -ct -cname -ip`

I will be writing a custom tool which consumes Stdin from the above command and parses the data into a MongoDB database. 

Using the amass database, I can continuously monitor for new discoveries. Eventually I will implement an alerting system using amass as the driver and Slack or Discord as the alert platform. 

---




## OSINT

This step consists of utilizing Github search and Shodan in order to find interesting hosts or content. Basically, the idea is to find things like login pages and whatnot, then try and search for things based on those in Github. So if you wanted to try and gather some default credential bugs, you would use some shodan queries

`ssl:"<org name>"" 200` -> org name having been found from the Horizontal Discovery step
`net:"<cidr>"` -> if a CIDR range is in scope for the target
`Ssl.cert.subject.CN:"example.com" 200` -> to search by domain name
`http.title:"Grafana" 200` -> then look through org facet for your organization. Can be anything sensitive, not just Grafana obviously

You can filter the shodan facets to tailor what you are looking for, and then use interesting things to inform your Github searches. 

This step is especially good for finding default credentials, interesting endpoints, IDORs and other secrets. 

Also, try some of these on every target: [https://www.exploit-db.com/google-hacking-database](https://www.exploit-db.com/google-hacking-database "https://www.exploit-db.com/google-hacking-database")

---




## Content Discovery
---

This is either the next step after Asset Discovery, or the next step after Initial Reconaissance. If it's not a wildcard scope you would also probably skip screenshotting and just try to discover content on the interesting subdomains that are in scope. Obviously, if you aren't going to screenshot, you should probably visit the subdomain and see what kind of authentication level you have or need before instantly ffuf'ing the target.

#### Screenshots

Again, probably skip this part if the target isn't a wildcard asset in scope. 

`# eyewitness -d target.com -f target.txt --threads 10`

Make sure Burpsuite is open. You will have it passively spider while you click on subdomains that look interesting from your screenshots. Look for things like: other full-function web applications, 403s to try to bypass, weird or custom looking apps or logins, etc. Anything interesting you use your custom ffuf list to try and discover content on, and add anything interesting to your list so you can keep curating it.

#### Directory Bruteforcing

Either you have your screenshots, or you have visited any interesting subomains manually. From here, anything that is a 403, an API, or anything else that looks like it could be hiding interesting content should be ffuf'd

`# ffuf -u <target_url> -c -D -t 100 -r -w <custom_wordlist>.txt`

Then start appending filters to content size or status codes you don't want to see until you've refined your search

---




## Hunting Phase
---

#### Authentication and Authorization

In this phase you will create a second account so you can try and take note of any differences or similariies in account IDs, see if there are different roles, try swapping one piece of account info for another, etc. (will add more to this later)

#### General Vuln Hunting

Since you have a full site map, you would then browse through burpsuite picking out interesting endpoint with parameterized requests both GET and POST, and based on what you see, try different vuln types. See a parameter that looks vulnerable to open redirect or SSRF? Maybe another one with a unique user ID to test for IDOR? What about this custom JS file? Maybe look for secrets or further endpoints in that. Perhaps a critical functionality caches requests that you could try to abuse. Maybe you could hit all the endpoints looking for indicators of HTTP desyncing, or H2C. This part is also where you could start identifying tools to write to automate some of this less-informed or low-hanging fruit hunting. This would give you a general idea of what potential things exist on this target, and you would start getting familiar with the attack surface as a whole. This is essentially still manual browsing, just in Burp instead of on the website, and you would associate website functionality with the actual underlying requests. 

**Payloads**
```
<s>000'")};--//
{{8*8}}[[5*5]]
- <%= `ls` %>
';--
'+OR+1=1--'
'%20OR%201=1--
"><img src=xss>
```

Drop those into every input and parameter you can find and check the response html and content size to look for changes. 

#### Deep Diving

Ok now that you've gained as much understanding of the target as you can, and possibly discovered or weeded out any low-hanging fruit, it's time for a deep dive. You would refer to your notes and look for specific functionality to test. You should be looking for more specific things based upon the functionality. Here's where you would be heavily driving whatever research you do after hours since you will inevitably run into the end of your current knowledge on particular functionalities. Once you've finished testing a piece of functionality you could say you thoroughly tested it and move on for good, or until your brain has one of those lightbulb moments.

#### Javascript Enumeration

- Download all JS files
	- Or finish the tool that extracts them all from source code (either way it's per domain, not per scope. )
	- Store them all in a database. Your tool could do this by scraping or by importing a text file containing all the JS files (full uri)
- Use all your regexes to extract interesting signatures from the JS source files
- Maybe extract all individual functions from the JS files so you can see all the function names
- Try to find the functionality that uses those signatures or functions.
	- Something something debugger blah blah

Don't forget that not all Javascript is in individual files. Some of it is just in the HTML. 


1) Download all JS. This isn't as straightforward as just going to Burpsuite and pulling down all the JS files underneath the host you are looking for, because that subdomain may not have any of it's own JS/only has it's JS inside of HTML/only includes JS from other subdomains. Some of those JS files don't belong to the company at all so you may not actually want them. I also don't necessarily just want to go to the subdomain that DOES host all the javascript because it might host JS files for other subdomains unrelated to the one you're deep-diving on (as is the case with static.gemini.com). So I guess what I could do is something like:
   
   ```go
   func downJsFiles(subdomain string) {
	   var regex = `regex`

		for _, x := range resp.Body {
			found := regex.FindString(x)
			
			if strings.Contains(found, subdomain) {
				fmt.Println(found)
			} else {
				// that js file name doesn't contain the subdomain we are scraping
			}
		}
   }
   ```

So in the case of exchange.gemini.com you've got these:
```html
<script type="text/javascript" src="https://static.gemini.com/js/runtime.ee068ebb900be29b20f3.js"></script>

<script type="text/javascript" src="https://static.gemini.com/js/0.801e26ec41f2fa3a6b02.js)"></script>

<script type="text/javascript" src="https://static.gemini.com/js/70.e7aebcc88f84c9b7ef8b.js"></script>

<script type="text/javascript" src="https://static.gemini.com/js/61.607002ffbc6c6f49001a.js"></script>```

And then you will have JS files from the HTML source requesting other JS files from within their source so

Also, deminifying files online using de4js or probably any other resource has the potential to freeze your browser if you paste a JS file that's too big in there, and many JS files are too big. Perhaps installing it locally is a solution

