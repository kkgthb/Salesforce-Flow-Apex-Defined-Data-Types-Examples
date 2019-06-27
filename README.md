# Salesforce Flow Apex-Defined Data Types (code examples)

Contained within is a bit more Apex code to support my blog [Katie Kodes](https://katiekodes.com).

This is to help Salesforce Developers play around with Apex-Defined Data Types for Salesforce Flow, as used for calling out to HTTP APIs, in a bit more depth than is presented at my companion blog article, which I recommend reading first:

> [Tutorial: Flow Apex-Defined Data Types for Salesforce Admins](https://katiekodes.com/flow-apex-defined-data-types/)

Many thanks to [Ultimate Courses's public API directory](https://github.com/public-apis/public-apis) for helping me find the 3 examples _([YesNo](#yesno), [Shouting As A Service](#shouting-as-a-service), and [Indian Cities](#indian-cities))_.

_(Want to help me add a 4th example?  See [my bug here](#still-debugging--zip-codes-for-a-us-city-new-62719) and [Tweet Me](https://twitter.com/KatieKodes) your thoughts.)_

## HTTPMockFactory class

You'll need this class for any of the examples.  _(Each example also has 3 more classes you need to create, including a unit test to run, and some Anonymous Apex to execute.)_

```java
public class HTTPMockFactory implements HttpCalloutMock {
    protected Integer code;
    protected String status;
    protected String body;
    protected Map<String, String> responseHeaders;
    public HTTPMockFactory(Integer code, String status, String body, Map<String, String> responseHeaders) {
        this.code = code;
        this.status = status;
        this.body = body;
        this.responseHeaders = responseHeaders;
    }
    public HTTPResponse respond(HTTPRequest req) {
        HttpResponse res = new HttpResponse();
        for (String key : this.responseHeaders.keySet()) {
            res.setHeader(key, this.responseHeaders.get(key));
        }
        res.setBody(this.body);
        res.setStatusCode(this.code);
        res.setStatus(this.status);
        return res;
    }
}
```

---

## YesNo

Return a random yes/no/maybe, and accompanying image, thanks to https://yesno.wtf/#api.

YesNo always returns JSON-formatted text representing _just one_ object with 3 properties, like this:

```json
{
    "answer":"no",
    "forced":false,
    "image":"https://yesno.wtf/assets/no/10-d5ddf3f82134e781c1175614c0d2bab2.gif"
}
```

### YesNo class

This class allows YesNo to be used as a Flow Variable data type.

```java
public class YesNo {
    @AuraEnabled @InvocableVariable public String answer;
    @AuraEnabled @InvocableVariable public Boolean forced;
    @AuraEnabled @InvocableVariable public String image;
}
```

### YesNoGenerator class

This class allows getYesNo(), a.k.a. "Get YesNo," to be used as an Invocable Apex Method in a Flow.

Flow seems to effectively "`[0]`" its return value and treat it as a single returned object -- hence declaring the return type of `getYesNo()` to be a `List` of `YesNo`s rather than a single `YesNo`.

```java
public class YesNoGenerator {
    @InvocableMethod(label='Get YesNo' description='Returns a response from the public API YesNo.wtf')
    public static List<YesNo> getYesNo() {
        List<YesNo> yesNo = new List<YesNo>{getRandomYesNo()};
        return yesNo;
    }
    
    private static YesNo getRandomYesNo() {
        YesNo yn = new YesNo();
        Http http = new Http();
        HttpRequest request = new HttpRequest();
        request.setEndpoint('https://yesno.wtf/api');
        request.setMethod('GET');
        HttpResponse response = http.send(request);
        // If the request is successful, parse the JSON response.
        if (response.getStatusCode() == 200) {
            // Deserialize the JSON string into collections of primitive data types.
            yn = (YesNo)JSON.deserialize(response.getBody(), YesNo.class);
        }
        return yn;
    }
}
```

### TestYesNoGenerator class

```java
@isTest
public class TestYesNoGenerator {
    static testMethod void testYesNoGenerator() {
        HttpMockFactory mock = new HttpMockFactory(
            200, 
            'OK', 
            '{'+
            '"answer":"no"'
            +','
            +'"forced":false'
            +','
            +'"image":"https://yesno.wtf/assets/no/10-d5ddf3f82134e781c1175614c0d2bab2.gif"'
            +'}', 
            new Map<String,String>()
        );
        Test.setMock(HttpCalloutMock.class, mock);
        Test.startTest();
        List<YesNo> returnedYNs = YesNoGenerator.getYesNo();
        Test.stopTest();
        System.assert(!returnedYNs.isEmpty());
        YesNo returnedYN = returnedYNs[0];
        System.assertEquals('no',returnedYN.answer);
    }
}
```


### Anonymous Apex for actually testing the yes/no/maybe API live

```java
System.assert(FALSE, YesNoGenerator.getYesNo()[0]);
```

You're expecting an error message with a Yes/No/Maybe and a GIF URL; something along the lines of:

```
Line: 1, Column: 1
System.AssertException: Assertion Failed: YesNo:[answer=yes, forced=false, image=https://yesno.wtf/assets/yes/3-422e51268d64d78241720a7de52fe121.gif]
```

If you get this error, you need to enable making callouts to https://yesno.wtf/api in your org's [Remote Site Settings](https://login.salesforce.com/one/one.app#/setup/SecurityRemoteProxy/home):

```
Line: 14, Column: 1
System.CalloutException: Unauthorized endpoint, please check Setup->Security->Remote site settings. endpoint = https://yesno.wtf/api
```

---

## SHOUTING AS A SERVICE

Uppercase your input text, thanks to http://shoutcloud.io

Shouting As A Service always returns JSON-formatted text representing _just one_ object with 2 properties, like this:

```json
{
    "INPUT":"helloWorld",
    "OUTPUT":"HELLOWORLD"
}
```

**Security warning:**  this service is not available over HTTPS.

It is _quite_ possible for anyone on the internet to intercept the response from Shouting As A Service and replace it with a virus before it gets back to you.

Likely?  Probably not.

But possible!  Play at your own risk.

### AllCaps class

This class allows AllCaps to be used as a Flow Variable data type.

```java
public class AllCaps {
    @AuraEnabled @InvocableVariable public String input;
    @AuraEnabled @InvocableVariable public String output;
}
```

### AllCapsGenerator class

This class allows getAllCaps(), a.k.a. "All-Caps Your Text (INSECURE HTTP ONLY)," to be used as an Invocable Apex Method in a Flow.

Again, flow seems to effectively "`[0]`" invocable methods' return values, so return a `List` of whatever you actually want to pass back to the flow.

Similarly, when invoked, it should be passed a single text-typed Flow Variable, not a "collection" / "multiple values"-enabled text-typed Flow Variable.

```java
public class AllCapsGenerator {
    @InvocableMethod(label='All-Caps Your Text (INSECURE HTTP ONLY)' description='Returns a response from the public API Shouting As A Service (INSECURE HTTP)')
    public static List<AllCaps> getAllCaps(List<String> inputText) {
        List<AllCaps> acText = new List<AllCaps>{getCapitalizedInputText(inputText[0])};
        return acText;
    }
    
    private static AllCaps getCapitalizedInputText(String inputString) {
        AllCaps ac = new AllCaps();
        Http http = new Http();
        HttpRequest request = new HttpRequest();
        request.setEndpoint('http://api.shoutcloud.io/V1/SHOUT'); // SECURITY VULNERABILITIES BECAUSE NOT AVAILABLE OVER HTTPS!  RUN AT YOUR OWN RISK!
        request.setMethod('POST');
        request.setHeader('Content-Type','application/json');
        request.setBody('{"INPUT":"'+inputString+'"}');
        HttpResponse response = http.send(request);
        // If the request is successful, parse the JSON response.
        if (response.getStatusCode() == 200) {
            // Deserialize the JSON string into collections of primitive data types.
            ac = (AllCaps)JSON.deserialize(response.getBody(), AllCaps.class);
        }
        return ac;
    }
}
```

### TestAllCapsGenerator class

```java
@isTest
public class TestAllCapsGenerator {
    static testMethod void testAllCapsGenerator() {
        String fixedInput = 'hello World';
        HttpMockFactory mock = new HttpMockFactory(
            200, 
            'OK', 
            '{'+
            '"INPUT": "hello World"'
            +','
            +'"OUTPUT": "HELLO WORLD"'
            +'}', 
            new Map<String,String>()
        );
        Test.setMock(HttpCalloutMock.class, mock);
        Test.startTest();
        List<AllCaps> returnedACs = AllCapsGenerator.getAllCaps(new List<String>{fixedInput});
        Test.stopTest();
        System.assert(!returnedACs.isEmpty());
        AllCaps returnedAC = returnedACs[0];
        System.assertEquals('HELLO WORLD',returnedAC.output);
    }
}
```

### Anonymous Apex for actually testing the shouting API live

```java
System.assert(FALSE, AllCapsGenerator.getAllCaps(new List<String>{('hi')})[0]);
```

You're expecting an error message of:

```
Line: 1, Column: 1
System.AssertException: Assertion Failed: AllCaps:[input=hi, output=HI]
```

If you get this error, you need to enable making callouts to http://api.shoutcloud.io/V1/SHOUT in your org's [Remote Site Settings](https://login.salesforce.com/one/one.app#/setup/SecurityRemoteProxy/home):

```
Line: 16, Column: 1
System.CalloutException: Unauthorized endpoint, please check Setup->Security->Remote site settings. endpoint = http://api.shoutcloud.io/V1/SHOUT
```

---

## Indian Cities

Return a random Indian city thanks to https://indian-cities-api-nocbegfhqg.now.sh/

Indian Cities always returns JSON-formatted text representing _a **list** of objects_ with 3 properties _apiece_, like this:

```json
[
    {
        "City":"Shansha",
        "State":"HimachalPradesh",
        "District":"Lahual"
    },
    {
        "City":"Kardang",
        "State":"HimachalPradesh",
        "District":"Lahual"
    }
]
```

### IndianCity class

This class allows IndianCity to be used as a Flow Variable data type.

Again, flow seems to effectively "`[0]`" invocable methods' return values, so return a `List` of whatever you actually want to pass back to the flow.

```java
public class IndianCity {
    @AuraEnabled @InvocableVariable public String city;
    @AuraEnabled @InvocableVariable public String state;
    @AuraEnabled @InvocableVariable public String district;
}
```

### IndianCityGenerator class

This class allows getIndianCity(), a.k.a. "Get Random Indian City," to be used as an Invocable Apex Method in a Flow.

```java
public class IndianCityGenerator {
    
    private static List<IndianCity> allIndianCities = new List<IndianCity>();
    
    @InvocableMethod(label='Get Random Indian City' description='Returns a response from the public Indian Cities API')
    public static List<IndianCity> getIndianCity() {
        List<IndianCity> indCit = new List<IndianCity>{getRandomIndianCity()};
            return indCit;
    }
    
    private static IndianCity getRandomIndianCity() {
        if ( allIndianCities.isEmpty() ) {
            Http http = new Http();
            HttpRequest request = new HttpRequest();
            request.setEndpoint('https://indian-cities-api-nocbegfhqg.now.sh/cities');
            request.setMethod('GET');
            HttpResponse response = http.send(request);
            // If the request is successful, parse the JSON response.
            if (response.getStatusCode() == 200) {
                // Deserialize the JSON string into collections of primitive data types.
                allIndianCities = (List<IndianCity>)JSON.deserialize(response.getBody(), List<IndianCity>.class);
            }
        }
        if ( allIndianCities.isEmpty() ) {
            return new IndianCity();
        } else {
            Integer randomNumber = Math.mod(Math.abs(Crypto.getRandomLong().intValue()),allIndianCities.size());
            return allIndianCities[randomNumber]; 
        }
    }
}
```

### TestIndianCityGenerator class

```java
@isTest
public class TestIndianCityGenerator {
    static testMethod void testIndianCityGenerator() {
        HttpMockFactory mock = new HttpMockFactory(
            200, 
            'OK', 
            '[{'+
            '"City":"Shansha","State":"Himachal Pradesh","District":"Lahual"}'+
            ','+
            '{"City":"Kardang","State":"Himachal Pradesh","District":"Lahual"'+
            '}]', 
            new Map<String,String>()
        );
        Test.setMock(HttpCalloutMock.class, mock);
        Test.startTest();
        List<IndianCity> returnedCities = IndianCityGenerator.getIndianCity();
        Test.stopTest();
        System.assert(!returnedCities.isEmpty());
        IndianCity returnedCity = returnedCities[0];
        System.assert(String.isNotEmpty(returnedCity.city));
        System.assert('Shansha|Kardang'.contains(returnedCity.city));
    }
}
```

### Anonymous Apex for actually testing the cities API live

```java
System.assert(FALSE, IndianCityGenerator.getIndianCity()[0]);
```

You're expecting an error message with a random city, something along the lines of:

```
Line: 1, Column: 1
System.AssertException: Assertion Failed: IndianCity:[city=Tuljapur, district=Osmanabad, state=Maharashtra]
```

If you get this error, you need to enable making callouts to https://indian-cities-api-nocbegfhqg.now.sh/cities in your org's [Remote Site Settings](https://login.salesforce.com/one/one.app#/setup/SecurityRemoteProxy/home):

```
Line: 17, Column: 1
System.CalloutException: Unauthorized endpoint, please check Setup->Security->Remote site settings. endpoint = https://indian-cities-api-nocbegfhqg.now.sh/cities
```


---

## Zip Codes for a U.S. City (new 6/27/19)

Inspired by Alex Edelstein's [official Salesforce video demo of Apex-Defined Data Types (from 12:53 to 19:29)](https://www.youtube.com/watch?v=oU0y38yf5qw&t=766) and his accompanying "Unofficial Salesforce" blog posts parts [1](https://unofficialsf.com/part-1-manipulate-complex-internet-data-in-flow-without-code/) & [2](https://unofficialsf.com/part-2-manipulate-complex-internet-data-in-flow-without-code/) and [video](https://www.youtube.com/watch?v=3xH1YLh5L7s), I decided to add one more example here that leverages Apex-Defined Data Types the way Salesforce _intended_ them to work:  with the parsing of "nested inner JSON classes" handed over to administrators within Flow, rather than being done by developers within Apex.

For this example, I chose an API from https://zippopotam.us that, provided a country like `us` and a state abbreviation like `wi` and a phrase like `beloit`, can search for all cities in that state containing the phrase `beloit` and return all zip codes assigned to Beloit, Wisconsin, USA.

Zippopotamus always returns JSON-formatted text with a more complex, nested structure than we've seen before, like this call to https://api.zippopotam.us/us/wi/beloit:

```json
{
  "country abbreviation": "US",
  "places": [
    {
      "place name": "Beloit",
      "longitude": "-89.086",
      "post code": "53511",
      "latitude": "42.562"
    },
    {
      "place name": "Beloit",
      "longitude": "-89.0728",
      "post code": "53512",
      "latitude": "42.6698"
    }
  ],
  "country": "United States",
  "place name": "Beloit",
  "state": "Wisconsin",
  "state abbreviation": "WI"
}
```

Of course, we could build a sophisticated tool that lets users enter their own city, state, and country, but there're a lot of issues to handle that I'd rather not go into.

For example, http://api.zippopotam.us/us/mn/Saint%20Paul returns data from 3 Minnesota cities containing `Saint Paul` in their names, the largest of which is Saint Paul itself, but attributes the overall list to the small suburb of Saint Paul Park:

```json
{
  "country abbreviation": "US",
  "places": [LOTS OF ZIPS FROM "Saint Paul Park," "South Saint Paul," "Saint Paul," etc.],
  "country": "United States",
  "place name": "Saint Paul Park",
  "state": "Minnesota",
  "state abbreviation": "MN"
}
```

Trying to search any city in France gets nasty quickly because the results are overwhelmingly filled with [CEDEX](https://en.wikipedia.org/wiki/Postal_codes_in_France#CEDEX) fake zip codes for businesses.

Obviously, this might not be the right API for you to use in real life!

To simplify things, in this example, we'll hard-code the "live" HTTP request to always ask about Mankato, Minnesota.

We'll do a "mock" in our test class with Moorhead, Minnesota just to make sure we didn't over-tailor our code to Mankato.

### CityZip class

This class allows CityZip to be used as a Flow Variable data type.

Be sure to avoid using underscores in the variable names -- Flow produces errors if you do.

```java
public class CityZip {
	@AuraEnabled @InvocableVariable public String countryabbreviation;
	@AuraEnabled @InvocableVariable public List<Place> places;
	@AuraEnabled @InvocableVariable public String country;
	@AuraEnabled @InvocableVariable public String placename;
	@AuraEnabled @InvocableVariable public String state;
	@AuraEnabled @InvocableVariable public String stateabbreviation;
}
```

### Place class

However, we also had a list of objects embedded in the JSON under the key `places`.

I decided to call an Apex class that could be used to represent one of these objects `Place`.  `CityZipPlace` would have done just as nicely, though.

`Place` needs to be an "outer," standalone Apex class, not an "inner," nested class inside `CityZip`, so that `Place` can be seen by Flow.

Be sure to avoid using underscores in the variable names -- Flow produces errors if you do.

```java
public class Place {
    @AuraEnabled @InvocableVariable public String placename;
    @AuraEnabled @InvocableVariable public String longitude;
    @AuraEnabled @InvocableVariable public String postcode;
    @AuraEnabled @InvocableVariable public String latitude;
}
```

### CityZipGenerator class

This class allows getCityZip(), a.k.a. "Get CityZip Mankato," to be used as an Invocable Apex Method in a Flow.

Again, flow seems to effectively "`[0]`" invocable methods' return values, so return a `List` of whatever you actually want to pass back to the flow.

Note our use of a Regular Expression replacement to eliminate any spaces in the JSON "keys," since we can't put spaces into the Apex-Defined Data Type attribute names to which we're deserializing.

_(At first I tried replacing the spaces with underscores for clarity, but it turns out Flow doesn't like variable names with underscores in them.)_

```java
public class CityZipGenerator {
    private static Pattern spaceInJSONKey = pattern.compile('(?<="[a-z]+) (?=[a-z]+": )');
    
    @InvocableMethod(label='Get CityZip Mankato' description='Returns a response about Mankato from the public API Zippopotam.us')
    public static List<CityZip> getCityZip() {
        List<CityZip> czs = new List<CityZip>{getMankatoCityZip()};
        return czs;
    }
    
    private static CityZip getMankatoCityZip() {
        CityZip cz = new CityZip();
        Http http = new Http();
        HttpRequest request = new HttpRequest();
        request.setEndpoint('https://api.zippopotam.us/us/mn/mankato');
        request.setMethod('GET');
        HttpResponse response = http.send(request);
        // If the request is successful, parse the JSON response.
        if (response.getStatusCode() == 200) {
            // Deserialize the JSON string into collections of primitive data types.
            cz = (CityZip)JSON.deserialize(
                spaceInJSONKey.matcher(response.getBody()).replaceAll(''),
                CityZip.class
            );
        }
        return cz;
    }
}
```

### TestCityZipGenerator class

```java
@isTest
public class TestCityZipGenerator {
    static testMethod void testCityZipGenerator() {
        HttpMockFactory mock = new HttpMockFactory(
            200, 
            'OK', 
            '{"country abbreviation": "US", "places": '+
            '[{"place name": "Moorhead", "longitude": "-96.7572", '+
            '"post code": "56560", "latitude": "46.8677"}, '+
            '{"place name": "Moorhead", "longitude": "-96.5062", '+
            '"post code": "56561", "latitude": "46.89"}, '+
            '{"place name": "Moorhead", "longitude": "-96.5062", '+
            '"post code": "56562", "latitude": "46.89"}, '+
            '{"place name": "Moorhead", "longitude": "-96.5062", '+
            '"post code": "56563", "latitude": "46.89"}], '+
            '"country": "United States", "place name": "Moorhead", '+
            '"state": "Minnesota", "state abbreviation": "MN"}', 
            new Map<String,String>()
        );
        Test.setMock(HttpCalloutMock.class, mock);
        Test.startTest();
        List<CityZip> returnedCZs = CityZipGenerator.getCityZip();
        Test.stopTest();
        System.assert(!returnedCZs.isEmpty());
        CityZip returnedCZ = returnedCZs[0];
        System.assertEquals('MN',returnedCZ.stateabbreviation);
        System.assertEquals('Moorhead',returnedCZ.placename);
        System.assert(returnedCZ.places[2].postcode.startsWith('5656'));
    }
}
```

### Anonymous Apex for actually testing the Zippopotam.us API live

```java
System.assert(FALSE, CityZipGenerator.getCityZip()[0]);
```

You're expecting an error message with information about Mankato, Minnesota, USA; something along the lines of:

```
Line: 1, Column: 1
System.AssertException: Assertion Failed: CityZip:[country=United States, countryabbreviation=US, placename=Mankato, places=(Place:[latitude=44.1538, longitude=-93.996, placename=Mankato, postcode=56001], Place:[latitude=44.056, longitude=-94.0698, placename=Mankato, postcode=56002], Place:[latitude=44.2172, longitude=-94.0942, placename=Mankato, postcode=56003], Place:[latitude=44.056, longitude=-94.0698, placename=Mankato, postcode=56006]), state=Minnesota, stateabbreviation=MN]
```

If you get this error, you need to enable making callouts to https://api.zippopotam.us in your org's [Remote Site Settings](https://login.salesforce.com/one/one.app#/setup/SecurityRemoteProxy/home) _(note that you can chop off `/us/mn/mankato`)_:

```
Line: 14, Column: 1
System.CalloutException: Unauthorized endpoint, please check Setup->Security->Remote site settings. endpoint = https://api.zippopotam.us/us/mn/mankato
```
