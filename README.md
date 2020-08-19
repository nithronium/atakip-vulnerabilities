# atakip-vulnerabilities
Proof of concept for vulnerabilities of Atakip.com website.

Atakip.com is a website that allows yellow cab owners to track their taxis' information online. The website provides GPS information for the cabs, their status, their speed, their earnings and information about the driver and owner of the cab. This repo includes proof of concept scripts for the vulnerabilities on the Atakip.com web address. 

### Type of the security vulnerabilities

Current vulnerability is classified as a software vulnerability that has design flaws. It is most probably caused by not having a security audit for the website or not being checked for possible attacks.

### What information could be obtained using the vulnerabilities?

Current vulnerabilities allows potential attackers to get full access to the user data on the website whether they are authorized or not. Those information include; the identification information of the driver and the owner of the cab, including their photo and Turkish Republic National Identification Number, the income for a time span of choice, the GPS locations for a time span of choice (GPS locations are recorded every 5 to 15 seconds), status of the cab (whether the engine is on or off, whether there is a customer or not, how fast it is going etc.)

The vulnerable backends return these datasets in JSON format so it is very easy for an attacker to data mine or process the provided datasets.

### How to reproduce the vulnerability?

#### 1- Gain access
In order to reproduce the vulnerability, an attacker has to have an access to a registered user on the system. The access to the portal is only provided to the people who owns a yellow cab and who is using the EN Taksi application (the main solution provider for the yellow cabs in Izmir, Turkey). An attacker could gain access by either finding a friend/relative who owns a yellow cab and has access to the system, or can brute-force since there is no re-captcha or similar application that blocks the automated requests and since the usernames are only at the form of 35TXXXX (where XXXX is a 4 digit number ranging from 5000 to 8000). Also most of the registered users use simple passwords like 123456 which makes the attackers job even easier.

In my case I was checking the information for my father's car, and only then realized the information was being passed visible to the eye.

#### 2- Send requests
After having the access to the portal, user is greeted with a website where they can perform basic queries to get information about their cab. It is exactly at that point where an attacker can easily start data mining. First, a user can send a request for seeing the GPS data of their car, which sends a request to the website **https://portal.entaksi.com.tr** 

#### 3- Modify the requests for data mining

After sending initial non-modified requests, attacker could listen the network activity and copy the previously sent non-modified request and modify it for their needs. Here is an example request for GPS data;

    await fetch("https://portal.entaksi.com.tr/AppService/GetCarTrack.do", {
        "credentials": "omit",
        "headers": {
            "User-Agent": "Mozilla/5.0 (X11; Ubuntu; Linux x86_64; rv:79.0) Gecko/20100101 Firefox/79.0",
            "Accept": "application/json, text/javascript, */*; q=0.01",
            "Accept-Language": "tr-TR,tr;q=0.9,en-US;q=0.8,en;q=0.7",
            "content-type": "application/x-www-form-urlencoded",
            "Pragma": "no-cache",
            "Cache-Control": "no-cache"
        },
        "body": "username=888&password=202CB962AC59075B964B07152D234B70&plateNumber=35TXXXX&starttime=2020-MM-DD+HH%3AMM%3ASS&endtime=2020-MM-DD+HH%3AMM%3ASS",
        "method": "POST",
        "mode": "cors"
    });
    
As you can see from the request, the body part of the POST request already includes a username, which is **888** and a password which is **202CB962AC59075B964B07152D234B70**. And by noticing the length of the password an attacker can easily realize that it's a possible MD5 hash and try to decrypt and actually find that the password is **123** hashed in MD5 

From now on, the attacker can modify the **plateNumber=35TXXXX** part for whatever plate number they would want to look up for (ranges from 5000 to 8000) and adjust the **starttime** and **endtime** for whatever time span they want.

Plus, there are some parts on the website where the body part is actually passed as a GET request with plain password in sight. Which is even a larger security threat.

#### 4- Response from the server

Such request will return a JSON object with the following parameters and data;

    errCode":0,
    "errMsg":"",
    "data":[
    {"direction":XXX,"gpslat":"XXX","gpslng":"XXX","gpstime":"XXX","speed":XXX,"state":XXX,"time":"XXX},
    {"direction":XXX,"gpslat":"XXX","gpslng":"XXX","gpstime":"XXX","speed":XXX,"state":XXX,"time":"XXX}]
    }
    
#### 5- Possible attack scenarios
    
Since modifying the plateNumber, starttime and endtime parameters on the body part of the initial request yields a valid response from the backend, an attacker could write an automated script to check all the data for all the yellow cabs for the Izmir province of Turkey. A simple javascript loop would allow an attacker to datamine and possibly sell & make profit from that mined data. As of now we don't know how large that dataset could be but it could easily be worth a few tens of thousands of dollars simply because a yellow cab can make near ₺30,000 a month and owning a plate number costs way over ₺1,200,000. 



### Additional Discoveries

Upon discovering the username and the password for the **https://portal.entaksi.com.tr** I have realized that one can easily gain access to the main portal and actually see all the information regarding any company that is operating under the EN Taksi platform. This is a huge vulnerability that could result in company losing a lot of money and especially leaking a lot of personal data of their clients. However, due to the national laws and regulations, accessing to that platform would be breaking the law so I haven't tried to do anything on that platform and stayed on the legal side. For that reason, it is unknown what one can exactly achieve with accessing to that platform.


### How it was discovered?

I was checking the network requests to find a way to download the GPS data for a yellow cab owned by my father. It was then I realized that the network request was being passed with username and password and URL in plain sight. I tried logging off from the website and checking the URL for my own yellow cab to actually see that the request still returned a valid data with no authentication. Later tried the same thing on a private browser on a different machine to verify the vulnerability was indeed there.

### How could it be fixed?

First of all, the main admin username and password must immediately change to something strong. Second of all, a recaptcha type of validation should be added for the main login area. Third, and most important of all, an authentication parameter should be added to each and every request on the website. 

Upon initial login, if a user successfully signs up, an authentication token is given to the user, registered on their local machinese as a cookie. The same token, exact date & time & IP address & their username is also storred on a seperate database table. From then, the authentication token is added to all the forms' POST & GET fields. And on the backend, each form is first checked for authentication token verification. And if the authentication token is not a valid one, the server must return a response with error. If the authentication token is a valid one, the server must return the username & IP address for that authentication token internally. After that response, another script checks the user's main request to match the datafields with the username.

For example above we have seen that modifying **plateNumber** variable allows possible attackers to download data for other usernames, a script that will check whether the authentication token is is registered for the plateNumber that is currently being requested or not. If the authentication token belongs to the same plate number as the user is requesting, then it will return a valid dataset response. However, if the user's authentication key's username doesn't match the one they are requesting from server, the request must be blocked. 

On top of that, there could be an IP matching for the authentication token and the request for not allowing any authentication token sharing between users but it is optional.


### Additional suggestions

It is a poor practice to pass parameters in a GET request if the master username and password has to be passed. Also the master username & password should never be passed. Instead, authentication token should be passed and authentication token should be validated for the requests instead of the master username and password. 

