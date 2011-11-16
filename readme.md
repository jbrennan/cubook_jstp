Introduction
============

The message protocol we've decided to use for this application is modeled after the most pervasive and successful message protocol of all time: the HyperText Transfer Protocol (HTTP). Our application uses a simplified version modeled after HTTP, using a smaller set of verbs, and using JSON instead of HTML for transmission of objects and commands.

Like HTTP, our protocol is stateless and session based. Each session can be one of the following: Request/Response or Notification based. Each will be explained in the subsequent sections.

There are two basic flows of events: the typical Request/Response, where a client requests some object (or updates an existing object) and the server responds with the requested resource; OR the Notification flow, where the Server sends a message to all clients (this message is essentially a Response that was not provoked by a Request).

JSTP
====

The basic premise of a JSTP session is the following:

1. 	A client makes a request to the server, creating a string-based *Header + Body*, with a *Verb* followed by the requested *Resource*, a user-name and password (plain-text), and the optional message body. Each *Header* line is separated by exactly 1 newline character, and the body, if any, is separated by an additional newline
	
		GET /boards/comp3004
		username: jbrennan
		password: sekret
		
	
2.	After the Server processes the Request, it then sends back a Response, in the form of a JSON object (described in full detail later in this document):
	
		{
			status: "OK",
			error: "",
			request: "/boards/comp3004"
			body: {
				// the returned object
			}
		}

3.	The server **might** then *push* out a similar message to all other clients, notifying them of any changes which just occurred (in the case of a POST from a client). This looks similar to a response:
	
		{
			status: "NOTIFY",
			error: "",
			request: "",
			body: {
				// newly pushed data
			}
		}

Verbs
-----

There are several verbs which may be used in both Requests and Responses (case insensitive). Each Verb takes as a parameter the URL *resource* which is being requested.


###GET

Requests from the Server a JSON object the given URL resource.
	
A request of this type should not contain a *message body* (i.e. the optional JSON object contained after the Header section of the Request). Any such body object will be ignored by the Server.
	
This request should have no side-effect on the server.
	
The Server *responds* with the requested object if the request was valid, otherwise it responds with a JSON object detailing the error.
	
	
###POST

This request **requires** there to be a message body after the headers, which contains a *new* or *updated* object to be placed at the location specified by the resource URL.
	
This request **should** have a side-effect on the server.

The Server responds with an appropriate JSON object on Success, or if there was an error, the JSON object will contain the error. The returned objects are defined below in the "Response" section, with each message body defined by the particular API.

###DELETE

This request deletes the specified resource, if and only if the user is authorized to do so.

This request **should** have a side-effect on the server.

The Server responds with an appropriate JSON object on Success, or if there was an error, the JSON object will contain the error. The returned objects are defined below in the "Response" section, with each message body defined by the particular API.


Resources
---------

A *Resource* is represented by a URL (without the host part) and points to something on the Server, be it an object, a collection of objects, or a command (these are defined by the API).

Resources are used in the Verb header of a request, and subsequently returned in the Response object to give the client application some context.

Resources should be surrounded by double quotes **and must not contain spaces**. Examples:

	"/users/jbrennan/info"
	"/courses/comp3004/newsflashes"
	


Requests
--------

A Request uses a set of headers and optionally a message body (depending on the Verb) to request from the Server the specified Resource (GET) or Command (POST, DELETE).

###Headers

These are the Headers required for each Request. Each header appears on its own line (i.e. separated by exactly 1 newline):

*	`VERB resource`: where VERB is one of `GET` or `POST` or `DELETE` and `resource` is a URL for the desired object or command.

*	`username: some_name`: where `some_name` is a valid user name for the system.

*	`password: some_password`: where `some_password` is the plaintext representation of the user's password.

###Message Body (for POST)

If the Request Verb is of type `POST` then there should also be a message body (in JSON) of the object(s) being updated or created. This message body should follow the Headers with an additional newline in between (so 2 newlines after the last header).

The object posted is up to the client and the Server's API.

###Examples

For GET:

	GET "/users/jbrennan/info"
	username: jbrennan
	password: sekret
	
For POST:

	POST "/boards/comp3004/newsflashes/new"
	username: jbrennan
	password: sekret
	
	{
		// this is up to the API
		newsflash-text: "A sample newsflash with an image %@",
		binaries: [
			"base64-image=="
		]
	}
	

Responses
---------

After a Server processes a Request (for any Verb type), it performs any necessary action (such as updating an object from a POST or removing one from a DELETE), and then returns a Response.

The response is in the form of a JSON object, which has the following key-value pairs:

		{
			status: "OK",
			error: "",
			request: "/boards/comp3004"
			body: {
				// the returned object
			}
		}

* `status`: The value for this key will be a string which will be one of the following:
	* `OK`: The request was good and the returned object(s) in the `body` key-value pair should be used by the client. The `error` key-value pair should be ignored in this case.
	* `Notification`: There was no request. Instead, this response was *pushed* to the client from the server. In this case, the `body` and `request` pairs are both valid and should be inspected to see how the client should react to the response.
	* `Error`: There was an error either with the request or with performing an associated action. The `error` pair should be **checked** to determine the cause of the error. The `body` pair should be ignored.
	
* `error`: The value for this key contain one of the following integer error codes (based on the HTTP error codes):
	* 200: OK. This is the value found when the `status` is "OK". The error shouldn't even be checked in such a condition, but this ensures no unexpected problems arise if it does get checked.
	* 400: Bad request. The client provided incorrect headers and the request could not be completed.
	* 403: Forbidden. The user credentials provided in the Request header were not sufficient for the user to complete the request (i.e. a Student trying to request another student's profile, they are not authorized to do this).
	* 404: Not found. The specified resource does not exist on the server.
	* 500: Internal Server Error. Useful for debugging the server.

* `request`: The URL of the requested resource, useful for providing context to the response when handling it.

* `body`: This is the returned JSON object for the request. It may be empty (i.e. `{}`). Otherwise, it will contain the JSON object as defined by the Server's API.


###Sending and receiving binary data (such as images)

When there is a need to send a binary file, such as an image file, the bytes of the file are encoded into a Base64 string and sent in a special way as follows:

1. Choose a key-value pair which will have binary files referenced in its value. A binary file reference looks like the following token: `%@`.
2. Create a second key-value pair whose key is `binaries` and whose value is an array of Base64 strings, each of which corresponds, in order, to a token in the above pair.
3. These will then be decoded by the receiving end and can be matched up where they belong.

This technique should be used not only in receiving binary files from the Server, but also sending them *to* the server as well.

Notifications
-------------

A notification is just like a Response from the Server, except it comes without having been Requested. This is a *Push* response, usually meaning another client has updated something and POSTed it to the Server, and the Server thought your client should know about it.

It is a returned object resembling a Response object with the exception of the `status` string being `Notification` instead of `OK`.


Network Datastructures (for Sockets)
------------------------------------

The raw packets are just `QString` which is turned into a `QByteArray` and sent over the TCP connection.


CUBook API
==========

The CUBook API is built on top of the the JSTP protocol with a similar mindset to a REST HTTP API. We will describe the different types of requests which can be made, any required object parameters, and also detail the corresponding Responses objects returned from the Server.

**Note**: Our protocol allows for user passwords, though they are not required according to the project requirements. Your server should then still cope with blank passwords sent by clients which don't deal with passwords.

Users
-----

###Checking log-in

	GET "/users/#username/login"

Where `#username` is a valid login name of your system. The way our protocol works, username and password are actually passed in every single request anyway, but this allows the client to verify two things:

1. If the username exists and if the password is valid.
2. If the user has previously logged in to the system before, or if they are a new user (new users can then display a Welcome screen, with the requirement to pick a nickname and optionally an image, as seen below).

This method returns an error if the user doesn't exist (`404`). Otherwise, it returns an object which looks like the following:

	{
		username: "jbrennan",
		nickname: "Jason",
		icon: "base64Image==",
	}
	
**Important**: If the value for `nickname` is null or the empty string, then the user has not yet chosen a nickname (i.e. this is their first log-in) and they must then pick a username before continuing.

###Registering a nickname and image

	POST "/users/#username/first_login"
	
	{
		nickname: "SomeDesiredNickname",
		icon: "base64Icon=="
	}

The above message body should be POST'ed with the request, sending along the input from the user. After this call is made, the First Login Window should not be dismissed and the user should not be able to use the application until a response has been received from the Server. Example response:

	{
		nickname-accepted: true
	}

If `nickname-accepted` is `true` then everything is good and the user may use the application. If the value was `false` then the user needs to pick a different nickname and try the process over again.

###User info

**Note**: In all example requests containing `#username`, this means whatever login name you use to find a user in your system (this is *not* the user's avatar/nickname).

	GET "/users/#username/info"

Where `#username` is some valid username in the system (ie their login info).

Returns a `403` error if the logged-in user is not authorized to see the requested user's information. Otherwise, it returns an object body in the following format (**note**: a Student can only see very limited information about a fellow student, and they can see full information about themselves. An instructor however can see the full information about any student taking his or her courses):

	{
		full-name: "Jason Brennan",
		nickname: "jbrennan",
		username: "jbrenna7@connect.carleton.ca",
		profile-image: "base64Image==",
		courses: [
			{
				// course info
			},
			{
				// course info
			},
			etc.
		],
		newsflashes: [
			{
				// newsflash info
			},
			{
				// newsflash info
			}, etc.
		]
	}

The `courses` and `newsflashes` array contain the same kind of objects as can be fetched using the APIs below for `course` or `newsflash`. See below for description of how these objects look.

###User Flashfeed

	POST "/users/#username/flashfeed"

This allows a client to retrieve the Flashfeed for the specified `#username`.

Because this method is a `POST` verb, the client can optionally provide a message body object to filter the returned flashfeed (i.e. which courses are included in the feed). If no message body is provided, then this method returns a Flashfeed containing newsflashes from all of the user's courses. If the feed should be filtered, the request should look like this:

	POST "/users/jbrennan/flashfeed"
	username: jbrennan
	password: sekret
	
	{
		courses: [comp3004, comp3000, ling2006]
	}
	
That is, it should only include courses which **should** appear in the flashfeed. Any courses not included will be omitted from the feed.

The returned object will resemble the following:

	{
		newsflashes: [
			// each item is a newsflash object as described below
		]
	}

FlashBoards
------

###Getting all newsflashes for a flashboard

	GET "/flashboards/#course/newsflashes"
	
Gets all the newsflashes for the Course specified by `#course`, so long as the user making the request is authorized to view the flashboard (responds with an error if the users is not authorized).

Returns an array of all the newsflashes in the given course if authorized, otherwise an error. Example message body:

	{
		flash-segments: [
			// described below
		]
	}
	

Each element in the `flash-segments` array will look like the following:

	{
		segment-id: "discuss1001",		// some unique identifier of any format
		segment-name: "Discussions",	// displayable name of the segment for your interface
		newsflashes: [
			// newsflash objects as described below
		]
	}

####Deleting a Flash-segment

	DELETE "/flashboards/#course/delete_segment"
	...login headers
	
	{
		segment-id: "discuss1001"
	}
	
This allows an authorized user (i.e. an instructor) to delete a Flash-segment. If the user is unauthorized, this API method returns an `error` of code `403` (Unauthorized), else it reponds with an `OK` status and an empty message body.

####Newsflash objects

Each newsflash object that appears in the API will look like the following:

	{
		newsflash-id: "123456"",
		newsflash-date: 54321542623
		newsflash-author-nick: "matilda"
		newsflash-author-icon: "base64string=="
		html-text: "This is a newflash with a <a href='http://google.com'>link</a> and an image %@",
		binaries: [
			"binaryImage1=="
		]
	}
	
`newsflash-id`, integer, is an indentifier for the newsflash object which may be useful to identify it uniquely, either for caching purposes or for storing it in a database.

`newsflash-date`, integer, is a Unix timestamp in seconds since Jan 1 1970, of when the newsflash was posted.

`html-text`, string, the text of the newsflash, including any links the user has included and tokens (`%@`) for any image files inline with the newsflash.

`binaries`, array of Base-64 encoded strings, each item in this array corresponds to a token (`%@`) in the `html-text` pair. Those tokens should be replaced with the properly decoded images found in this array.

###Posting a Newsflash

	POST "/flashboards/#course/add_newsflash"

This post requires an object like the following message body in the Request:

	{
		html-text: "This is a newflash with a <a href='http://google.com'>link</a> and an image %@",
		binaries: [
			"binaryImage1=="
		]
	}

This method returns the exact same newsflash object on success, or an error on failure (for example, if the user was not authorized to post a newsflash to the specified flashboard).

###Deleting a Newsflash

	DELETE "/flashboards/#course/delete_newsflash"
	...login headers
	
	{
		newsflash-id: "someid1234"
	}

This method deletes a user's newsflash so long as they are authorized to do so. Responds with an error if the user is not authorized. On success, the message body is empty (i.e. `{}`).

###Creating a Flashsegment

	POST "/flashboards/#course/addflashsegment"

Adds a new Flash-Segment to the given course flashboard. The new object must be posted with the request like in the following example:

	POST "/flashboards/comp3004/addflashsegment"
	username: jbrennan
	password: sekret
	
	{
		flash-segment-name: "Study help"
	}

The reposnse will contain an error if the user was not authorized, or on success, the `status` will be `OK` and the `body` will just be `{}`.

Courses
-------

###Course info

This is the same info as can be found in the user info method above. Example:

	{
		course-id: "comp3004",			// an identifier for the course
		course-name: "Comp 3004",		// A displayable string for the course
		shown-in-flashfeed: true		// a boolean to indicate whether or not the user has set this course to be shown in the flashfeed
	}

###Course shown in Flashfeed

The user should be allowed to filter their Flashfeed to only show newsflashes from their prefered courses. This list must be preserved accross sessions of the app. To do this, when the user updates their preference, post this change to the server:

	POST "/users/#username/flashfeed/toggle_course"
	...login headers
	
	{
		course-id: "comp3004",		// The course to be toggled
		shown-in-flashfeed: false	// sad panda :(
	}

