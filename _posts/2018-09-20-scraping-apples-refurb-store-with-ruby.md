---
layout: post
title: "Scraping Apple's Refurb Store with Ruby"
date: 2018-09-20 09:00 -0500
categories: ruby apple iphone gmail gem nokogiri parsing regex cronjob cron crontab
---

Last week Apple announced their new lineup of iPhones, just like they do every year. This time though, it seems like they're pushing to make $1000+ smartphones the norm. They've also completely removed fingerprint authentication from their phones in favor of Face ID, which they claim is more secure. If you're like me and you don't want to spend an absurd amount of money on a phone, and want fingerprint authentication, chances are that you're better off purchasing an older, marked-down iPhone model.  

If you want to save even more money (like myself) and still want a premium Apple warranty, you buy refurbished! The nice thing about refurbished Apple products is that they are in essence, products that were originally in great condition – either returned by unhappy customers or pulled from Apple's display units in stores. The refurbishment process includes swapping the old battery for a new one, wiping the phone clean and replacing parts that had cosmetic damage. So what you end up getting is basically a brand new device for cheap. The thing about these refurbished devices though, is that people seem to jump on them very quickly, so they sell out just as fast. What we are going to do is devise a system that will notify us when the iPhone 8 is available for purchase on Apple's Refurbished Store. Let's get started! 

I'm going to be using Ruby to scrape the Apple Store site, simply because the use of gems and the syntax makes it very easy to perform this action in a minimal amount of code. The gems that I am going to be using are 'open-uri' to access files on the web, 'nokogiri' to collect information from HTML pages and 'gmail' to create a notification system. 

### 1. Set up

First, install all the gems that you'll need for this to work. I am using the 'gmail' gem to send myself email notificaitons if the search returns a positive hit for the product that I am searching for. In this case, it's the iPhone 8. If you don't care to integrate email notifications into your implementation of this, you can omit that part. 
```shell
sudo gem install open-uri
sudo gem install nokogiri
sudo gem install gmail
```

### 2. Basic Layout

Since I am going to throw this on a server and I want the file to be interpreted as a Ruby script, I will omit the _.rb_ extension from my filename. I'll want to store the URL that I want to go to as a variable. I will also store the product that I want to search for as a variable. Since we are going to be scraping surface level text on the website and not going into the sitemap, we can just use the product name as-is as our search term. In other words, I know *exactly* what I want to search for, and I know exactly how it will appear on the page. 

I am also going to fetch the contents of the page using Nokogiri and store them as a string in another variable. 

Lastly, I am going to set up my conditional block to process something if the text that I grabbed from the web page contains my search term.

```ruby
#!/usr/bin/env ruby

require 'open-uri'
require 'nokogiri'
require 'gmail'
# the URL for the page that shows refurbished iPhones
URL = "https://www.apple.com/shop/browse/home/specialdeals/iphone" 
# your search term variable
iphone_8 = "iPhone 8"

page = Nokogiri::HTML(open URL)
content = page.to_s

if content.include? iphone_8
 # do something here
end

exit 0
```

### 3. Integrating Email Notifications

In the case that you don't want to set up notifications, you can omit this next step completely and just use a `puts` or equivalent to print a message to the screen. That works just fine, but you don't get notified in real time when the script returns the search result. If I know anything about Apple's history of putting iPhones on the refurbished store, its that they sell like hotcakes! So I am also going to implement a notification system that integrates with my personal Gmail, letting me know when the product I'm searching becomes available.

The `connect` method provided by the gmail gem allows me to make a connection to my Google Account and access my email. I can also specify my recipient(s) and email details such as the subject and body, if applicable. You can check out the gmail gem in more detail on [it's landing page on Github](https://github.com/gmailgem/gmail). When I have constructed and `deliver`ed the email, I will also want to logout and close the connection to my Google Account, to prevent a dangling open connection.

```ruby
...
...
if content.include? iphone_8
  Gmail.connect('me@gmail.com', 'yourpassword') do |gmail|
    gmail.deliver do 
      to "also_me@gmail.com"
      subject "Product XYZ Is Now Available!"
      text_part do 
        body "#{iphone_8} is now available in the Apple Refurbished store!\n Hurry on over to #{URL} to grab one now!"
      end
    end
    gmail.logout
  end
end
```

There are a couple of things to note here that I think are worth mentioning:
- using your Gmail credentials in plain text is obviously **NOT** an ideal solution. If you're going to put your code on some sort of publicly available repository, you'll want to make sure that your credentials are hidden. For that, I suggest checking out the BCrypt gem and XOauth2 to create a more secure authentication method. In the interest of time, I have omitted this step.
- you'll want to make sure that you allow your Google Account to [Let Less Secure Apps access your account](https://myaccount.google.com/lesssecureapps). This is basically a security feature that Google has disabled by default for account security. To turn it on, please go to the above link to turn it on. The `gmail` gem won't work unless you turn this on.

### 4. Automating The Process Using Cron

Like I said before, if you want to just manually run your script you should be able to do that at this point. But if you want to automate and schedule the task to run ever so often, we can use `CRON` to set that up. CRON is a UNIX utility that allows you to schedule tasks at a regular time interval. The task in this case is our script. This way, after a specified time the script will automatically run, parse the website and send me an email if the search term is found.

For this, you can use a service like Heroku, AWS or DigitalOcean, but since I have a Raspberry Pi lying around, I will just use that as my server. 

Setting up CRON is pretty straight forward. The three commands that you'll need to know are:
```shell
crontab -l #lists all cronjobs
crontab -e #create a new cronjob
crontab -r #remove cronjob(s)
```

When you launch a new cronjob using the `-e` flag, you'll see a screen that gives you some basic information about how to set up a job. It should be pretty self explanatory, but to be clear, cronjobs are set up with this format:
```shell
# MM HH DD MO WK
* * * * * /path/to/task/to/run -arguments 
```
The `*` is a wildcard that in this case means 'run the script every minute of every hour of every day of every month indefinitely'. You can use a web resource called [crontab guru](https://crontab.guru) to double check the schedule that you want your task to follow.

So let's say I want to run my script every 4 hours indefinitely. That seems like a pretty decent time interval for scraping the page. For that, I would write:
```shell
0 */4 * * * /bin/bash -l -c '~/path/to/filename'
```
This means 'run the file at the specified path from the bash environment at the 0th minute every 4 hours'. Pretty neat, right? Go ahead and test it out by searching for something that's already there on the page you're trying to scrape and see if the cronjob actually works. If it does, you're all done!

---

We can see that by using a few lines of code and by utilizing a couple of rubygems and CRON, we can set up a system that scrapes a particular web resource and notifies us when something that we're interested on that resource becomes available. That seems like a very powerful implementation and a pretty big win to me! Now, when the iPhone becomes available on the refurb store, and depending on how frequently you set up your cronjob to execute, you can be pretty darn sure that you'll be one of the first ones to be notified.

Cheers,

– rehmanh

