### Engage your mobile users with Progressive Web Apps built on IBM Bluemix

In this lab, you will learn how to

* Create a mobile application that you can also host as a web app from a single source. 
* Host the mobile web app on IBM Cloud (IBM Bluemix) using Node.js Runtime on IBM Cloud
* Make your web app into a Progressive Web App
* Interact with a backend service from within your mobile and web app using IBM Mobile Foundation

#### Pre-requisites
1. A Bluemix Account - Create one [here](https://console.bluemix.net/) 
2. Bluemix CLI - Download from [here](https://clis.ng.bluemix.net/ui/home.html)
3. Node.js Runtime - Download from [here](https://nodejs.org/en/download/)

#### Setup 
##### 1. Install Cordova 

> 	npm install -g cordova

##### 2. Set up your Bluemix Command line 

First set the region and Bluemix endpoint
> bluemix api https://api.ng.bluemix.net

Login to your IBM Bluemix account using the following command
> bluemix login -u < username > -o < organization > -s < space >

You can get the space and organization from the top right corner of the bluemix console


#### Create your mobile and web app

Download the MobileFirst sample app
> md C:\ISTC2017 

> cd C:\ISTC2017

> git clone https://github.com/MobileFirst-Platform-Developer-Center/PinCodeCordova
> 
> cd PinCodeCordova

Add the `android` platform for mobile apps and the `browser` platform for web apps 

> cordova platform add android
> 
> cordova platform add browser

Add the IBM MobileFirst SDK For Cordova to the app 

> cordova plugin add cordova-plugin-mfp

Now, you have an app that's coded for both the Mobile and Web. The android platform will generate a project that can be used to build an Android app. The browser platform will do the same for a web app. 

At this juncture, your app is ready to be previewed. The web app still cannot connect to a backend, but you can see the user interface. 

Preview your app

> cordova run browser 

This will open a new browser window with your web app's UI. This is the same UI that will be available on a Android/iOS/Windows app as well. 

#### Hosting your app on IBM Bluemix
After you've created your app and previewed it locally, its time to take it to the cloud. IBM Bluemix provides a Node.js runtime. Let us now host your web app on the Node.js runtime. 

1. Login to your Bluemix account. Go to [https://bluemix.net](https://bluemix.net) 
2. Click on Catalog -> Cloud Foundry Apps -> SDK For Node.js
3. Enter a unique app name and click Create 

We'll start by hosting the Node starter app and then host our app.

Download the Node Starter App 
>cd C:\ISTC2017
>
>git clone https://github.com/IBM-Bluemix/get-started-node
>
>cd get-started-node
>
>bluemix app push < app name > 
>(This is the same app name you provided in step 3 above)

Quickly verify by visiting https://< appname >.mybluemix.net from your browser.

Navigate to the cordova browser project folder 
> cd c:\ISTC2017\PinCodeCordova\platforms\browser\www

Copy the contents to `C:\ISTC2017\get-started-node\views`
With these changes, push your new app to Bluemix to ensure the changes are reflected.
>bluemix app push < appname >

Open your app in a browser using the same url as earlier and you will see the PinCode app is now live. 

#### Make your app a PWA
In this lab, we will demonstrate two of the multi-faceted capabilities of progressive web apps.

1. Serving content offline 
2. Accessing the app from the home screen 


**Serving Content Offline**

To serve web content offline, we make use of a HTML5 concept called service worker. A service worker is simply a piece of JavaScript code that the browser executes in the background independent of the web page that is calling this. This makes service workers multi-threaded too. 
Let us now add a service worker to our web app.


Navigate to the `C:\ISTC2017\get-started-node\views` folder and create a new file here called `service-worker.js`

**Note: Ensure the file is saved as a js file and not a text file**

Add the following content to it

```
var cacheName = 'MyAppCache';
var filesToCache = [
  '/',
  '/index.html',
  '/js/index.js',
  '/css/index.css',
  '/style.css'
];

/* Install the service worker */
self.addEventListener('install', function(e) {
  console.log('[ServiceWorker] Install');
  e.waitUntil(
    caches.open(cacheName).then(function(cache) {
      console.log('[ServiceWorker] Caching app shell');
      return cache.addAll(filesToCache);
    })
  );
});


/* Activate */
self.addEventListener('activate', function(e) {
  console.log('[ServiceWorker] Activate');
  e.waitUntil(
    caches.keys().then(function(keyList) {
      return Promise.all(keyList.map(function(key) {
        if (key !== cacheName) {
          console.log('[ServiceWorker] Removing old cache', key);
          return caches.delete(key);
        }
      }));
    })
  );
  return self.clients.claim();
});

/* Fetch */
self.addEventListener('fetch', function(e) {
  console.log('[ServiceWorker] Fetch', e.request.url);
  e.respondWith(
    caches.match(e.request).then(function(response) {
      return response || fetch(e.request);
    })
  );
});
```
Add the following code to your `index.html` within the `<head>` tag

```
<script type="text/javascript">
	if ('serviceWorker' in navigator){
		navigator.serviceWorker.register('./service-worker.js')
		.then(function() { console.log('Service worker registered successfully');});
	}
</script>
```
This will ensure that your browser is compatible with HTML5 service workers and then registers a service worker for your domain. 

With these changes, push your new app to Bluemix
>bluemix app push < appname >

You can now test the offline capabilities of your PWA by switiching off the internet and accessing your web app through the browser. 

**Access your app from the home screen**

Open the web app from a mobile browser (Chrome / Firefox / Opera) . Click on the settings menu and choose 'Add to Home Screen'

You can now access your web app from the home screen of a device just like a native app. You can also further verify the offline capabilities by switching off the internet on the mobile and accesing the app.

Additionally, meeting certain criteria, browsers automatically prompt users to add your web app to the home screen. 

#### Let your PWA do the work - access a mobile backend

For this section, we will use IBM Mobile Foundation on Bluemix as your mobile gateway. For convenience, we have already setup a Mobile Foundation instance with the necessary adapters deployed. Adapters are server side code that will fetch data from a backend and provide it to the mobile app or web app. 

Add the following to your `index.js` under `wlInitOptions` 

```   
mfpContextRoot : '/mfp', // "mfp" is the default context root in the MobileFirst Development server
applicationId : 'com.sample.pincodecordova' 

```

Also, replace your `server.js` with the following code 

```
var express = require("express");
var app = express();
var cfenv = require("cfenv");
var bodyParser = require('body-parser')
var cors = require('cors');
var http = require('http');
var fs = require('fs'); 
app.use(cors());
app.options('*', cors());


var httpProxy = require('http-proxy');
var apiProxy = httpProxy.createProxyServer();

var request = require('request');
// MFP url, unless set in npm start arg, this is set to localhost:9080
var mfpURL = process.argv[2] || 'https://istc-labea-server.mybluemix.net:443';

// parse application/x-www-form-urlencoded
app.use(bodyParser.urlencoded({ extended: false }))

// parse application/json
app.use(bodyParser.json())
//serve static file (index.html, images, css)
app.use(express.static(__dirname + '/views'));
 })


app.get('/mfp/*', function(req, res) {
  var url = mfpURL + req.originalUrl;
  req.pipe(request(url)).pipe(res);
});

app.post('/mfp/*', function(req, res) {
  var url = mfpURL + req.originalUrl;
  var headers = {};
  headers['Accept'] = req.headers['Accept'];
  headers['Content-type'] = req.headers['Content-type'];
  headers['x-mfp-analytics-metadata'] = req.headers['x-mfp-analytics-metadata'];
  headers['x-wl-analytics-tracking-id'] = req.headers['x-wl-analytics-tracking-id'];
  // headers['x-wl-analytics-tracking-id'] = req.headers['x-wl-analytics-tracking-id'];


  var options = {
    url: url,
    headers: headers,
    postData:JSON.stringify(req.body)
  };
   
  function callback(error, response, body) {
    if (!error && response) {
      response.pipe(res);
    }else{
      console.log('Error')
    }
  }
   
  request(options, callback);
});

var port = process.env.PORT || 3000
app.listen(port, '0.0.0.0', function() {
    console.log("To view your app, open this link in your browser: http://localhost:" + port);
});
```

The above code is needed so that your web app knows where to redirect MobileFirst requests to. 

Push your changes to Bluemix again 
> bluemix app push < appname >

Now, if you click on the `Get Balance` button, the app will connect to an IBM Mobile Foundation instance running on IBM Bluemix which will fetch data from a backend. 

#### Conclusion:
In this lab, you have learnt how to write an app with common code for mobile and the web. 
You also learnt how to make your web app a progressive web app and how to leverage the robustness of your mobile infrastructure to serve mobile web apps as well


#### Annexure:
** Setting up IBM Mobile Foundation

* Login to your Bluemix account 
* Go to Catalog -> Web and Mobile -> IBM Mobile Foundation 
* Create an instance of IBM Mobile Foundation 
* Login to the MobileFirst console
* Create a new Application - provide a name, select Web as the platform and `com.sample.pincodecordova` as the application ID
* Build and deploy the sample adapters from https://github.com/MobileFirst-Platform-Developer-Center/PinCodeCordova

