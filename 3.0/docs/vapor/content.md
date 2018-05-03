# Using Content

In Vapor 3, all content types (JSON, protobuf, [URLEncodedForm](../url-encoded-form/getting-started.md), [Multipart](../multipart/getting-started.md), etc) are treated the same. All you need to parse and serialize content is a `Codable` class or struct.

For this introduction, we will use mostly JSON as an example. But keep in mind the API is the same for any supported content type.

## Server

This first section will go over decoding and encoding messages sent between your server and connected clients. See the [client](#client) section for encoding and decoding content in messages sent to external APIs.

### Request

Let's take a look at how you would parse the following HTTP request sent to your server.

```http
POST /login HTTP/1.1
Content-Type: application/json

{
    "email": "user@vapor.codes",
    "password": "don't look!"
}
```

First, create a struct or class that represents the data you expect. 

```swift
import Vapor

struct LoginRequest: Content {
    var email: String
    var password: String
}
```

Notice the key names exactly match the keys in the request data. The expected data types also match. Next conform this struct or class to `Content`.

#### Decode

Now we are ready to decode that HTTP request. Every [`Request`](#fixme) has a [`ContentContainer`](#fixme) that we can use to decode content from the message's body.

```swift
router.post("login") { req -> Future<HTTPStatus> in
    return req.content.decode(LoginRequest.self).map { loginRequest in
        print(loginRequest.email) // user@vapor.codes
        print(loginRequest.password) // don't look!
        return HTTPStatus.ok
    }
}
```

We use `.map(to:)` here since `decode(...)` returns a [future](../async/getting-started.md). 

!!! note
    Decoding content from requests is asynchronous because HTTP allows bodies to be split into multiple parts using chunked transfer encoding. 

#### Router

To help make decoding content from incoming requests easier, Vapor offers a few extensions on [`Router`](#fixme) to do this automatically.

```swift
router.post(LoginRequest.self, at: "login") { req, loginRequest in
    print(loginRequest.email) // user@vapor.codes
    print(loginRequest.password) // don't look!
    return HTTPStatus.ok
}
```

#### Detect Type

Since the HTTP request in this example declared JSON as its content type, Vapor knows to use a JSON decoder automatically. This same method would work just as well for the following request.

```http
POST /login HTTP/1.1
Content-Type: application/x-www-form-urlencoded

email=user@vapor.codes&don't+look!
```

All HTTP requests must include a content type to be valid. Because of this, Vapor will automatically choose an appropriate decoder or error if it encounters an unknown media type.

!!! tip
    You can [configure](#configure) the default encoders and decoders Vapor uses.
    
#### Custom

You can always override Vapor's default decoder and pass in a custom one if you want.

```swift
let user = try req.content.decode(User.self, using: JSONDecoder())
print(user) // Future<User>
```

### Response

Let's take a look at how you would create the following HTTP response from your server.

```http
HTTP/1.1 200 OK
Content-Type: application/json

{
    "name": "Vapor User",
    "email": "user@vapor.codes"
}
```

Just like decoding, first create a struct or class that represents the data that you are expecting.

```swift
import Vapor

struct User: Content {
    var name: String
    var email: String
}
```

Then just conform this struct or class to `Content`. 

#### Encode

Now we are ready to encode that HTTP response.

```swift
router.get("user") { req -> User in
    return User(name: "Vapor User", email: "user@vapor.codes")
}
```

This will create a default `Response` with `200 OK` status code and minimal headers. You can customize the response using a convenience [`encode(...)`](#fixme) method.

```swift
router.get("user") { req -> Future<Response> in
    return User(name: "Vapor User", email: "user@vapor.codes")
        .encode(status: .created)
}
```

#### Override Type

Content will automatically encode as JSON by default. You can always override which content type is used
using the `as:` parameter.

```swift
try res.content.encode(user, as: .urlEncodedForm)
```

You can also change the default media type for any class or struct.

```swift
struct User: Content {
    /// See `Content`.
    static let defaultContentType: MediaType = .urlEncodedForm

    ...
}
```

## Client

Encoding content to HTTP requests sent by [`Client`](#fixme)s is similar to encoding HTTP responses returned by your server. 

### Request

Let's take a look at how we can encode the following request.

```http
POST /login HTTP/1.1
Host: api.vapor.codes
Content-Type: application/json

{
    "email": "user@vapor.codes",
    "password": "don't look!"
}
```

#### Encode

First, create a struct or class that represents the data you expect.

```swift
import Vapor

struct LoginRequest: Content {
    var email: String
    var password: String
}
```

Now we are ready to make our request. Let's assume we are making this request inside of a route closure, so we will use the _incoming_ request as our container. 

```swift
let loginRequest = LoginRequest(email: "user@vapor.codes", password: "don't look!")
let res = try req.client().post("https://api.vapor.codes/login") { loginReq in
    // encode the loginRequest before sending
    try loginReq.content.encode(loginRequest)
}
print(res) // Future<Response>
```

### Response

Continuing from our example in the encode section, let's see how we would decode content from the client's response.

```http
HTTP/1.1 200 OK
Content-Type: application/json

{
    "name": "Vapor User",
    "email": "user@vapor.codes"
}
```

First of course we must create a struct or class to represent the data.

```swift
import Vapor

struct User: Content {
    var name: String
    var email: String
}
```

#### Decode

Now we are ready to decode the client response.

```swift
let res: Future<Response> // from the Client

let user = res.flatMap { try $0.content.decode(User.self) }
print(user) // Future<User>
```

### Example

Let's now take a look at our complete [`Client`](#fixme) request that both encodes and decodes content.

```swift
// Create the LoginRequest data
let loginRequest = LoginRequest(email: "user@vapor.codes", password: "don't look!")
// POST /login
let user = try req.client().post("https://api.vapor.codes/login") { loginReq in 
    // Encode Content before Request is sent
    return try loginReq.content.encode(loginRequest) 
}.flatMap { loginRes in
    // Decode Content after Response is received
    return try loginRes.content.decode(User.self) 
}
print(user) // Future<User>
```

## Query String

URL-Encoded Form data can be encoded and decoded from an HTTP request's URI query string just like content. All you need is a class or struct that conforms to [`Content`](#fixme). In these examples, we will be using the following struct.

```swift
struct Flags: Content {
     var search: String?
     var isAdmin: Bool?
}
```

### Decode

All [`Request`](#fixme)s have a [`QueryContainer`](#fixme) that you can use to decode the query string.

```swift
let flags = try req.query.decode(Flags.self)
print(flags) // Flags
```

### Encode

You can also encode content. This is useful for encoding query strings when using [`Client`](#fixme).

```swift
let flags: Flags ...
try req.query.encode(flags)
```

## JSON

JSON is a very popular encoding format for APIs and the way in which dates, data, floats, etc are encoded is non-standard. Because of this, Vapor makes it easy to use custom [`JSONDecoder`](#fixme)s when you interact with other APIs.

```swift
// Conforms to Encodable
let user: User ... 
// Encode JSON using custom date encoding strategy
try req.content.encode(json: user, using: .custom(dates: .millisecondsSince1970))
```

You can also use this method for decoding.

```swift
// Decode JSON using custom date encoding strategy
let user = try req.content.decode(json: User.self, using: .custom(dates: .millisecondsSince1970))
```

If you would like to set a custom JSON encoder or decoder globally, you can do so using [configuration](#configure).

## Configure

Use [`ContentConfig`](#fixme) to register custom encoder/decoders for your application. These custom coders will be used anywhere you do `content.encode`/`content.decode`.

```swift
/// Create default content config
var contentConfig = ContentConfig.default()

/// Create custom JSON encoder
var jsonEncoder = JSONEncoder()
jsonEncoder.dateEncodingStrategy = .millisecondsSince1970

/// Register JSON encoder and content config
contentConfig.use(encoder: jsonEncoder, for: .json)
services.register(contentConfig)
```