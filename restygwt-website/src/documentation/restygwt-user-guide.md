# RestyGWT User Guide

RestyGWT is a GWT generator for REST services and JSON encoded data transfer objects.

{:toc:2-5}


## Features

* Generates Async Restful JSON based service proxies
* Java Object to JSON encoding/decoding
* Easy to use REST API

## REST Services

RestyGWT Rest Services allow you to define an asynchronous Service API which is then implemented via
GWT deferred binding by a RestyGWT generator.  See the following listing for an example:

{pygmentize::java}
import javax.ws.rs.POST;
...
public interface PizzaService extends RestService {
    @POST
    public void order(PizzaOrder request, 
                      MethodCallback<OrderConfirmation> callback);
}
{pygmentize}

JAX-RS annotations are used to control how the methods interface map to HTTP requests.  The 
interface methods MUST return void.  Each method must declare at least one callback argument.  
Methods can optionally declare one method argument before the callback to pass via the request
body.

Java beans can be sent and received via JSON encoding/decoding.  Here what the classes declarations
look like for the `PizzaOrder` and `OrderConfirmation` in the previous example:

{pygmentize::java}
public class PizzaOrder {
    public String phone_number;
    public boolean delivery;
    public List<String> delivery_address = new ArrayList<String>(4);
    public List<Pizza> pizzas = new ArrayList<Pizza>(10);
}

public class OrderConfirmation {
    public long order_id;
    public PizzaOrder order;
    public double price;
    public Long ready_time;
}
{pygmentize}

The JSON encoding style is compatible with the default [Jackson](http://wiki.fasterxml.com/JacksonHome) Data Binding.  En example,
`PizzaOrder` JSON representation would look like:

{pygmentize::js}
{
  "phone_number":null,
  "delivery":true,
  "delivery_address":[
    "3434 Pinerun Ave.",
    "Wesley Chapel, FL 33734"
  ],
  "pizzas":[
    {"quantity":1,"size":16,"crust":"thin","toppings":["ham","pineapple"]},
    {"quantity":1,"size":16,"crust":"thin","toppings":["extra cheese"]}
  ]
}
{pygmentize}

A GWT client creates an instance of the REST service and associate it with a HTTP
resource URL as follows:

{pygmentize::java}
Resource resource = new Resource( GWT.getModuleBaseURL() + "pizza-service");

PizzaService service = GWT.create(PizzaService.class);
((RestServiceProxy)service).setResource(resource);

service.order(order, callback);
{pygmentize}

### Request Dispatchers

The request dispatcher intercepts all requests being made and can supply
additional features or behavior.  For example, RestyGWT also supports a 
`CachingRetryingDispatcher` which will automatically retry requests if 
they fail.

To configure the `CachingRetryingDispatcher`, you can configure it on
your service interface at either the class or method level.  Example:

{pygmentize::java}
    @POST
    @Options(dispatcher=CachingRetryingDispatcher.class)
    public void order(PizzaOrder request, 
                      MethodCallback<OrderConfirmation> callback);
{pygmentize}

### Creating Custom Request Dispatchers

You can create a custom request dispatcher by implementing the following `Dispatcher`
interface:

{pygmentize::java}
public interface Dispatcher {
    public Request send(Method method, RequestBuilder builder) throws RequestException;
}
{pygmentize}

Your dispatcher implementation must also define a static singleton instance in a public
`INSTANCE` field.  Example:

{pygmentize::java}
public class SimpleDispatcher implements Dispatcher {
    public static final SimpleDispatcher INSTANCE = new SimpleDispatcher();

    public Request send(Method method, RequestBuilder builder) throws RequestException {
        return builder.send();
    }
}
{pygmentize}

When the dispatcher's `send` method is called, the provided builder will already
be configured with all the options needed to do the request.

### Configuring the Request Timeout

You can use the @Options annotation to configure the timeout for requests
at either the class or method level.  The timeout is specified in milliseconds,
For example, to set a 5 second timeout:

{pygmentize::java}
    @POST
    @Options(timeout=5000)
    public void order(PizzaOrder request, 
                      MethodCallback<OrderConfirmation> callback);
{pygmentize}


### Configuring the Expected HTTP Status Code

By default results that have a 200, 201, or 204 HTTP status code are considered
to have succeeded. You can customize these defaults by setting the 
@Options annotations at either the class or method level of the service interface.

Example:

{pygmentize::java}
    @POST
    @Options(expect={200,201})
    public void order(PizzaOrder request, 
                      MethodCallback<OrderConfirmation> callback);
{pygmentize}

### Configuring the `Accept` and `Content-Type` and HTTP Headers

RestyGWT rest calls will automatically set the `Accept` and `Content-Type` 
and HTTP Headers to match the type of data being sent and the type of
data expected by the callback, you you can override these default values
by adding JAXRS `@Produces` and `@Consumes` annotations to the method 
declaration.

### Mapping to a JSONP request

If you need to access JSONP URl, then use the @JSONP annotation on the method
for example:

{pygmentize::java}
import org.fusesource.restygwt.client.JSONP;
...
public interface FlickrService extends RestService {  
    @Path("http://www.flickr.com/services/feeds/photos_public.gne?format=json")
    @JSONP(callbackParam="jsonFlickrFeed")
    public void photoFeed(JsonCallback callback);    
}
{pygmentize}



## JSON Encoder/Decoders
    
If you want to manually access the JSON encoder/decoder for a given type just define
an interface that extends JsonEncoderDecoder and RestyGWT will implement it for you using
GWT deferred binding.

Example:

First you define the interface:
 
{pygmentize::java}
import javax.ws.rs.POST;
...
public interface PizzaOrderCodec extends JsonEncoderDecoder<PizzaOrder> {
}
{pygmentize}

Then you use it as follows
 
{pygmentize::java}
// GWT will implement the interface for you
PizzaOrderCodec codec = GWT.create(PizzaOrderCodec.class);

// Encoding an object to json
PizzaOrder order = ... 
JSONValue json = codec.encode(order);

// decoding an object to from json
PizzaOrder other = codec.decode(json);
{pygmentize}

### Customizing the JSON Property Names 

If you want to map a field name to a different json property name, you
can use the `@Json` annotation to configure the desired name.  Example:

{pygmentize::java}
public class Message {
    @Json(name="message-id")
    public String messageId;
}
{pygmentize}

## REST API

The RestyGWT REST API is handy when you don't want to go through the trouble of creating 
service interfaces.

The following example, will post  a JSON request and receive a JSON response. 
It will set the HTTP `Accept` and `Content-Type` and `X-HTTP-Method-Override` header t
o the expected values.  It will also expect a HTTP 200 response code, otherwise it will 
consider request the request a failure.

{pygmentize::java}
Resource resource = new Resource( GWT.getModuleBaseURL() + "pizza-service");

JSONValue request = ...

resource.post().json(request).send(new JsonCallback() {
    public void onSuccess(Method method, JSONValue response) {
        System.out.println(response);
    }
    public void onFailure(Method method, Throwable exception) {
        Window.alert("Error: "+exception);
    }
});
{pygmentize}

All the standard HTTP methods are supported: 

* `resource.head()`
* `resource.get()`
* `resource.put()`
* `resource.post()`
* `resource.delete()`

Calling one of the above methods, creates a `Method` object.  You must create a new one 
for each HTTP request.  The `Method` object uses fluent API for easy building
of the request via method chaining.  For example, to set the user id and password
used in basic authentication you would:

    method = resource.get().user("hiram").password("pass");

You can set the HTTP request body to a text, xml, or json value.  Unless the content type
is explicitly set on the method, the following methods will set the `Content-Type` header 
for you:

* `method.text(String data)`
* `method.xml(Document data)`
* `method.json(JSONValue data)`

The `Method` object also allows you to set headers, a request timeout, and the expected 
'successful' status code for the request.

Finally, once you are ready to send the http request, pick one of the following methods
based on the expected content type of the result.  Unless explicitly set on the method, 
the following methods will set the `Accept` header for you:

* `method.send(TextCallback callback)`
* `method.send(JsonCallback callback)`
* `method.send(XmlCallback callback)`

The response to the HTTP request is supplied to the callback passed in the `send` method.
Once the callback is invoked, the `method.getRespose()` method to get the GWT `Response`
if your interested in things like the headers set on the response.
