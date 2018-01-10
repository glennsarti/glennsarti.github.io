---
title: Aussie Broadband Usage script
excerpt: A quick PowerShell script to get your current broadband usage using web scraping
category:
  - blog
header:
  overlay_image: /images/header-powershell-banner.png
  overlay_filter: 0.50
  teaser: /images/teaser-psmabb.png
tags:
  - powershell
  - script
---

{% include toc icon="tags" %}

[Source Code](https://github.com/glennsarti/PSMyAussieBroadband)

---

So I've moved back to Australia and I needed an internet connection.  I settled on the ISP [Aussie Broadband](https://www.aussiebroadband.com.au/) (ABB).  But I didn't know how much data I needed!!  "All of it!" is the normal answer, but not fiscally responsible.

So I needed to keep track of data usage.  However, ABB doesn't have a mobile app or even an API to query.  I _does_ have a web portal you can log into and view your usage.  But this become tiresome after a while, and missed a bit of data I needed; How much data do I need to get through to the next billing cycle.  So, it was time to flex some PowerShell muscles.

So firstly I needed a nice way to show the usage.  I remember seeing a PowerShell module to draw graphs in the console; and sure enough [Prateek Singh](https://github.com/PrateekKumarSingh) wrote [PSConsoleGraph](https://github.com/PrateekKumarSingh/PSConsoleGraph).  Unfortunately this hasn't been posted as a module on the gallery yet (**HINT HINT**) so I just cloned the source.

### Authenticating

Then it was just a case of firing up Chrome Developer tools and watching the HTTPS traffic flow.  It seems the first call to the usage page (https://my.aussiebroadband.com.au/usage.php) expects a login form with username and password and returns a session cookie.  And then the next call to the usage page, with a valid session cookie, returns the data we need. A little bit of PowerShell and tada I have my usage statistics!

``` powershell
$body = @{
  'login_username' = $username
  'login_password' = $password
  'submit' = 'Login'
}

# Get a session token
$result = Invoke-WebRequest -Uri $usageURI -Method POST -Body $body -SessionVariable mabbSession -UseBasicParsing
# Now query for real...
$result = Invoke-WebRequest -Uri $usageURI -Method POST -Body $body -SessionVariable mabbSession
```

### Web Scraping

So we now have the HTML page but how do we extract the data?

The data is in a table;

![Usage Table]({{ site.url }}/blog-images/aussie_usage.png)

So we just need to find the relevant table and search through all of the table data (TD) elements.  In particular I needed to find the table that has a header with the text 'Internet Data'.  As PowerShell has access to the HTML DOM, this was trivial to do!

One thing that was annoying.  Every now and then the DOM parsing would just fail with weird error codes.  After a bit of searching I found this article [https://www.sepago.com/blog/2016/05/03/powershell-exception-0x800a01b6-while-using-getelementsbytagname-getelementsbyname](https://www.sepago.com/blog/2016/05/03/powershell-exception-0x800a01b6-while-using-getelementsbytagname-getelementsbyname).  Turns out I need to use the fully qualified method name `IHTMLDocument3_getElementsByTagName` and then everything was happy!!

### Data Output

Once I had the relevant table, it was just a case of working through each table row and extract the data I needed - Date, Data Downloaded and Data Uploaded.  I then created a simple array which had a total data for each day

``` powershell
@(11.21, 2.89, 8.82 ... )
```

... and then output that as a graph

``` powershell
Show-Graph -Datapoints $dataPoints -XAxisTitle 'Date' -YAxisTitle 'Total' -YAxisStep 1
```

That gave me prior usage but was still short of what I wanted.  A little more web scripting and I figured out the total usage for the billing cycle and how much data I had left.  I also figured out the dates of the billing cycle.

So now with a bit of maths;

1. We divide the Data Used by the number of hours elapsed in the current billing cycle.  This will give us an average Data per hour (Gb/h)
2. We then calculate the projected data I need, by multiplying the average Data per hour by the total number of hours in the billing cycle; and then subtract the data already consumed.
3. We can then compare the projected data I need, against how much data is left in the broadband plan

Then we can output this information in the script with a red colour if we don't have enough data and green if we do, for example;

![Console Usage Table]({{ site.url }}/blog-images/aussie_ps_usage.png)

### Storing Credentials

I was happy with what I had so far but if I wanted to share this script, I couldn't hardcode usernames and passwords.  So I turned to the [PSSecret](https://www.powershellgallery.com/packages/PSSecret/1.0.0) module by [Kiran Reddy](https://github.com/v2kiran).  I then created a new script `Invoke-MyAussieBroadbandusage.ps1` which managages loading and saving credentials.  So now you could either supply the username and password as parameters to `Get-MyAussieBroadbandUsage.ps1` or use the Invoke script to handle this as well.

## Wrap Up

After an hour or so I could query my broadband usage via a single command in PowerShell! Pretty nice.

Feel to try it out yourself with the [Source Code](https://github.com/glennsarti/PSMyAussieBroadband).

---

## Reference

### Get-MyAussieBroadbandUsage.ps1

#### Parameters

* `-Username` : Username to log into the My Aussie portal

* `-Password` : Password to log into the My Aussie portal

#### Output

Displays the broadband usage for your current billing cycle

![Example Image]({{ site.url }}/blog-images/aussie_ps_usage.png)

It also displays how much data you have left, and how much data you'll probably need by the end of your billing cycle (Red colour is bad, Green is good!)


### Invoke-MyAussieBroadbandusage.ps1

#### Parameters

* `-Username`        : Username to log into the My Aussie portal

* `-Password`        : Password to log into the My Aussie portal

* `-SaveCredentials` : A Switch that when enabled will save your credentials for later use

The `Invoke-MyAussieBroadbandusage` script will load saved credentials if none were passed, otherwise the Username and Password parameters will take precedence.  By default credentials will not saved.

The script will then run the `Get-MyAussieBroadbandUsage.ps1` script with the supplied credentials.

Invoke-MyAussieBroadbandusage can be thought of as a credential wrapper around the Get-MyAussieBroadbandUsage script.

``` powershell
PS> .\Invoke-MyAussieBroadbandusage.ps1 -Username Glenn -Password Foo -SaveCredentials

... Shows the usage graph

PS> .\Invoke-MyAussieBroadbandusage.ps1

... Shows the usage graph

PS>
```

The first time the script is called we save the credentials and the second time we call the script we no longer have to pass username or password.