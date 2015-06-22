# Couchbase by Example: Displaying a Sync Progress Indicator

`NSProgress` is an object in Foundation that represents the completion of some work. That work could be downloading a file, installing an app or something your own application is doing.

The NSProgress exists to let you easily report progress in your application across various components both for the UI and the system.

## Tutorial

Imagine you are on a vacation far from home and you need a simple application to browse the interesting places to visit in a city while sightseeing. You will use the Google Places API to import the restaurants in London in Sync Gateway.

> https://developers.google.com/places/

Then, you will use a pull replication to save all locations to the iOS app for offline use. The replication could take some time, you will use the NSProgress API to display a visual indicator to the user.

## Creating a Google API project

To use the Google Places API in this tutorial, you will first create a new project in the Google Developer Console and then generate a Server API Key.

Open the Google Developer Console and log into your account:

>  https://console.developers.google.com

Create a new Project called `City Explorer`:

![][image-1]

Once the new project appears in the list, click on it and navigate to `APIs & auth > APIs` in the left navigation drawer. Enable the `Google Places API Web Service`.

Once enabled, go to the `Credentials` tab in the left navigation drawer, create a new Key (Server key) and copy down the API key as you will need it throughout this tutorial.

In the next section, you will use a couple libraries and the Admin REST API to sync the Places data to Sync Gateway.

## Sync Gateway

Download Sync Gateway and unzip the file: 

> http://www.couchbase.com/nosql-databases/downloads#Couchbase\_Mobile

You can find the Sync Gateway binary in the `bin` folder and examples of configuration files in the `examples` folder. Copy the `basic-walrus-bucket.json` file to the root of your project:

	$ cp /Downloads/couchbase-sync-gateway/examples/basic-walrus-bucket.json /path/to/proj/sync-gateway-config.json

Start Sync Gateway:

	$ ~/Downloads/couchbase-sync-gateway/bin/sync_gateway

Open the Admin Dashboard to monitor the documents that were saved to Sync Gateway.

	http://localhost:4985/_admin/

In the next section, you will code a small NodeJS app with the RxJS and Request modules to import the Places data in Sync Gateway.

## Places API → Sync Gateway

Before you start scripting the app server, check that your API Key is working correctly, open the following url in your browser, you should see the JSON response.

	https://maps.googleapis.com/maps/api/place/textsearch/json?query=restaurants+in+London&key=API_KEY

**NOTE**: Don’t forget to add your API Key in the URL.

![][image-2]

To build the app server that will import the data from the Places API to Sync Gateway, you will use [RxJS][1] and [Request][2]. Code that deals with more than one event or asynchronous computation gets complicated quickly. RxJS makes these computations first-class citizens and provides a model that allows for readable and composable APIs to deal with these asynchronous computations. The Request module is the de-facto library to make http requests in NodeJS simpler than ever. Go ahead and install the dependencies:

	npm install request rx --save

Copy `requestRx.js` from the GitHub repo to your project folder. We’re simply wrapping the Request api in RxJS constructs (flatMap, filter, subscribe...). For exampe, instead of using `request.get`, you will use `requestRx.get`.

Create a new file called `sync.js`, require the `requestRx` and `Rx` modules. Define a couple constants:

	const api_key = 'AIzaSyBGRQzQ2Sy1zgIrMrbYUknd1L25idYOoII';
	const url = 'https://maps.googleapis.com/maps/api/place';
	const gateway = 'http://localhost:4985/db';

Next, use the `requestRX` command to follow the chain of requests describe in the diagram.

If you are wondering how to use Reactive Extensions, I stronly encourage you to follow this tutorial. It will take a couple hours to complete but you will come out of it with a very clear understanding of Reactive Extensions:

	http://reactive-extensions.github.io/learnrx/

This might be a lot to take in but the best think to do is experiment with the different operators (flatMap, zip, subscribe, fromArray):

	// 1. Search for Places
	requestRx.get(`${url}/textsearch/json?key=${api_key}&query=restaurants+in+london`)
	  .subscribe((res) => {
	      var places = JSON.parse(res.body).results;
	      var placesStream = Rx.Observable.fromArray(places);
	
	      // 2. Send the Places in bulk to Sync Gateway
	      requestRx({uri: `${gateway}/_bulk_docs`, method: 'POST', json: {docs: places}})
	        .flatMap((docsRes) => {
	            var docsStream = Rx.Observable.fromArray(docsRes.body);
	
	            // Merge the place's photoreference with the doc id and rev
	            return Rx.Observable.zip(placesStream, docsStream, (place, doc) => {
	                return {
	                    id: doc.id,
	                    rev: doc.rev,
	                    ref: place.photos[0].photo_reference
	                }
	            });
	        })
	        .flatMap((doc) => {
	
	            // 3. Get the binary jpg photo using the ref property (i.e. photoreference)
	            var options = {
	                uri: `${url}/photo?key=${api_key}&maxwidth=400&photoreference=${doc.ref}`,
	                encoding: null
	            };
	            return requestRx.get(options)
	              .flatMap((photo) => {
	
	                  // 4. Save the photo as an attachment on the corresponding document
	                  return requestRx({
	                      uri: `${gateway}/${doc.id}/photo?rev=${doc.rev}`,
	                      method: 'PUT',
	                      headers: {'Content-Type': 'image/jpg'},
	                      body: photo.body
	                  })
	              })
	        })
	        .subscribe((res) => {
	        });
	  });

1. Get the Places that match the query `restaurants in London`. Use the ES 6 string interpolation feature in the url.
2. The `_bulk_docs` endpoint is very convenient for importing large datasets in you instance of Sync Gateway. Read more about it in the [docs][3].
3. After saving the document, we save the photo as an attachment, we must first get the image from the Places API. Notice the `encoding` property is set to null. This is required by the Request module for any response body that isn’t a string. Read more about it in the [Request docs][4].
4. We must tell Sync Gateway which document (by specifying the document id) and revision to save this attachment on.

To run your NodeJS app written with the JavaScript ES 6 syntax, you can use the babel module. Install it and run it with the sync.js file:

	$ npm install babel -g
	$ babel-node sync.js

![][image-3]

Now that you have documents including images stored in the in-memory bucket of Sync Gateway, you will start coding the iOS app to include a progress bar managed by the replication change notification.

## iOS application

In Xcode, create a new Single View Application called `CityExplorer`:

![][image-4]

Close the project and install the Couchbase Lite iOS SDK via Cocoapods:

	$ pod init
	$ pod search couchbase

Add the dependency to the `Podfile` in the root of the project, then run install:

	$ pod install

Open `CityExplorer.workspace` this time and create a bridging header:

![][image-5]

Navigate to build settings to add the bridging header:

![][image-6]

Open `ViewController.swift` and add a property `pull` of type `CBLReplication?`. In the `viewDidLoad:` method, instantiate the replication and the register as an observer on the `kCBLReplicationChangeNotification`:

	let manager = CBLManager.sharedInstance()
	let databaseExists = manager.databaseExistsNamed("cityexplorer")
	var database = manager.databaseNamed("cityexplorer", error: nil)
	if databaseExists {
	    database?.deleteDatabase(nil)
	    database = manager.databaseNamed("cityexplorer", error: nil)
	}
	
	let gateway = NSURL(string: "http://localhost:4984/db")!
	
	pull = database?.createPullReplication(gateway)
	
	let nctr = NSNotificationCenter.defaultCenter()
	nctr.addObserver(self, selector: "replicationProgress:", name: kCBLReplicationChangeNotification, object: pull)
	
	pull?.start()
 

And implement the `replicationProgress` method to simply log the changesCount and completedChangesCount properties:

	func replicationProgress(notification: NSNotification) {
	    println("Changes count \(pull?.changesCount)")
	    println("Completed \(pull?.completedChangesCount)")
	}

In the next section, you will add a progress view in the Storyboard, then will connect it to the View Controller.

## Progress Bar

In the Storyboard, add a Progress View in the centre of the View:

![][image-7]

Connect the UI handle to the controller property:

![][image-8]

In the replication change listener, update the progressView's `progress` property accordingly:

	let active = pull?.status == .Active
	let completed = pull?.status == .Stopped
	
	println("Status : \(pull?.status.rawValue)")
	println("Changes Count: \(pull?.changesCount)")
	println("Completed Count: \(pull?.completedChangesCount)")
	println("======")
	
	if pull!.changesCount > 0 {
	    let number = Float(pull!.completedChangesCount) / Float(pull!.changesCount)
	    self.progressView.progress = number
	}

Run the app and you should see the progress bar updating as the documents are replicated:

![][image-9]

You can run the `sync.js` script a couple times to have more documents to import. The Places API returns a maximum of 20 results in one request.

## Conclusion

In this tutorial, you learned how to set up a Project to use the Google Places API and a NodeJS program to import data as documents and attachments in Sync Gateway. You also used the `NSNotification` api on iOS to register for the `kCBLReplicationChangeNotification` notification and update the progress view in your iOS application.

[1]:	https://github.com/Reactive-Extensions/RxJS
[2]:	https://github.com/request/request
[3]:	http://developer.couchbase.com/mobile/develop/references/sync-gateway/rest-api/database/post-bulk-docs/index.html
[4]:	https://github.com/request/request#user-content-requestoptions-callback

[image-1]:	http://i.gyazo.com/83a703f1a7330e42f7a92f62023c4daf.gif
[image-2]:	http://cl.ly/image/1X3A1S3x180b/sync_progress_indicator.png
[image-3]:	http://i.gyazo.com/5ff13132ea63ec95299165ab2868f4f8.gif
[image-4]:	http://cl.ly/image/0b2x1E2I012W/Screen%20Shot%202015-06-21%20at%2017.44.52.png
[image-5]:	http://i.gyazo.com/cc51fa04f3f3ae421bfee381e283e7b0.gif
[image-6]:	http://i.gyazo.com/2993e437948f8001a351a92b511e843a.gif
[image-7]:	http://cl.ly/image/0d330e3R3r3R/Screen%20Shot%202015-06-21%20at%2019.24.17.png
[image-8]:	http://cl.ly/image/2S01342L2933/Screen%20Shot%202015-06-21%20at%2019.29.53.png
[image-9]:	http://i.gyazo.com/148bfa61a0f5b8c1ba2fa3e09c5f807e.gif