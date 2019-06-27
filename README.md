# Salesforce Flow Apex-Defined Data Types (code examples)

Contained within is a bit more Apex code to support my blog [Katie Kodes](https://katiekodes.com).

This is to help Salesforce Developers play around with Apex-Defined Data Types for Salesforce Flow, as used for calling out to HTTP APIs, in a bit more depth than is presented at my companion blog article, which I recommend reading first:

> [Tutorial: Flow Apex-Defined Data Types for Salesforce Admins](https://katiekodes.com/flow-apex-defined-data-types/)

Many thanks to [Ultimate Courses's public API directory](https://github.com/public-apis/public-apis) for helping me find the 3 examples _([YesNo](#yesno), [Shouting As A Service](#shouting-as-a-service), and [Indian Cities](#indian-cities))_.

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

## STILL DEBUGGING:  Zip Codes for a U.S. City

### PROBLEM:

I can't quite get this one to actually work in a Flow.  I'm trying to imitate [Salesforce's official video demo of Apex-Defined Data Types (from 12:53 to 19:29)](https://www.youtube.com/watch?v=oU0y38yf5qw&t=766) and Alex Edelstein's accompanying "Unofficial Salesforce" blog posts parts [1](https://unofficialsf.com/part-1-manipulate-complex-internet-data-in-flow-without-code/) & [2](https://unofficialsf.com/part-2-manipulate-complex-internet-data-in-flow-without-code/) and [video](https://www.youtube.com/watch?v=3xH1YLh5L7s), but I just cannot, for the life of me, actually get this Apex-Defined Data Type working in a Flow like the others.

In theory, I should be able to create an Apex Action in Flow that calls `CityZipGenerator.getCityZip()` _(a.k.a. `Get CityZip Mankato`)_ and assigns the output to a `CityZip`-typed Apex-Defined Variable called `myCity`, with the following subcomponents:

- `country` gets mapped to `{!myCity.country}`
- `country_abbreviation` gets mapped to `{!myCity.country_abbreviation}`
- `place_name` gets mapped to `{!myCity.place_name}`
- `places` gets mapped to `{!myCity.places}`
- `state` gets mapped to `{!myCity.state}`
- `state_abbreviation` gets mapped to `{!myCity.state_abbreviation}`

I should be able to set up a Screen with Display Text rendering `{!myCity.state_abbreviation}` and, when I run my flow, I should see this:

```
MN
```

But instead, I see this:

```
{!{
"state_abbreviation" : "MN",
"state" : "Minnesota",
"places" : [ {
"post_code" : "56001",
"place_name" : "Mankato",
"longitude" : "-93.996",
"latitude" : "44.1538"
}, {
"post_code" : "56002",
"place_name" : "Mankato",
"longitude" : "-94.0698",
"latitude" : "44.056"
}, {
"post_code" : "56003",
"place_name" : "Mankato",
"longitude" : "-94.0942",
"latitude" : "44.2172"
}, {
"post_code" : "56006",
"place_name" : "Mankato",
"longitude" : "-94.0698",
"latitude" : "44.056"
} ],
"place_name" : "Mankato",
"country_abbreviation" : "US",
"country" : "United States"
}.{state_abbreviation}}
```

Even if I remove this mapping from the Apex Action's output:

- `places` to `{!myCity.places}`

I still get this when I run the Flow:

```
{!{
"state_abbreviation" : "MN",
"state" : "Minnesota",
"place_name" : "Mankato",
"country_abbreviation" : "US",
"country" : "United States"
}.{state_abbreviation}}
```

The "Debug Details" for this flow don't look terribly different from a successful "Yes/No/Maybe" flow...

```
CITYZIPGENERATOR (APEX): Grab_Zips
Inputs:
None.
Outputs:
{!myCity.state} = state (Minnesota)
{!myCity.place_name} = place_name (Mankato)
{!myCity.state_abbreviation} = state_abbreviation (MN)
{!myCity.country} = country (United States)
{!myCity.places} = places ([Place : { "post_code" : "56001", "place_name" : "Mankato", "longitude" : "-93.996", "latitude" : "44.1538" },Place : { "post_code" : "56002", "place_name" : "Mankato", "longitude" : "-94.0698", "latitude" : "44.056" },Place : { "post_code" : "56003", "place_name" : "Mankato", "longitude" : "-94.0942", "latitude" : "44.2172" },Place : { "post_code" : "56006", "place_name" : "Mankato", "longitude" : "-94.0698", "latitude" : "44.056" }])
{!myCity.country_abbreviation} = country_abbreviation (US)
```

```
YESNOGENERATOR (APEX): Get_YesNo
Inputs:
None.
Outputs:
{!aYesNo.answer} = answer (yes)
{!aYesNo.image} = image (https://yesno.wtf/assets/yes/8-2f93962e2ab24427df8589131da01a4d.gif)
{!aYesNo.forced} = forced (false)
```

Fellow devs:  any thoughts on why `myCity` is gacking when `aYesNo` is not?

### Background

Return all zip codes for a U.S. city thanks to https://zippopotam.us

_(In this case, we'll hard-code the "live" HTTP request to always ask about Mankato, Minnesota, rather than dealing with cleaning up user input.  We'll do a "mock" in our test class with Moorhead, Minnesota just to make sure we didn't over-engineer for the city of Mankato.)_

Zippopotamus always returns JSON-formatted text with a more complex, nested structure than we've seen before, like this call to https://api.zippopotam.us/us/wi/beloit indicating the 2 ZIP codes used for the city of Beloit, Wisconsin, USA:

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

### CityZip class

This class allows CityZip to be used as a Flow Variable data type.

```java
public class CityZip {
	@AuraEnabled @InvocableVariable public String country_abbreviation;
	@AuraEnabled @InvocableVariable public List<Place> places;
	@AuraEnabled @InvocableVariable public String country;
	@AuraEnabled @InvocableVariable public String place_name;
	@AuraEnabled @InvocableVariable public String state;
	@AuraEnabled @InvocableVariable public String state_abbreviation;
}
```

### Place class

However, we also had a list of objects embedded in the JSON under the key `places`.

I decided to call an Apex class that could be used to represent one of these objects `Place`.  `CityZipPlace` would have done just as nicely, though.

`Place` needs to be an "outer," standalone Apex class, not an "inner," nested class inside `CityZip`, so that `Place` can be seen by Flow.

```java
public class Place {
    @AuraEnabled @InvocableVariable public String place_name;
    @AuraEnabled @InvocableVariable public String longitude;
    @AuraEnabled @InvocableVariable public String post_code;
    @AuraEnabled @InvocableVariable public String latitude;
}
```

### CityZipGenerator class

This class allows getCityZip(), a.k.a. "Get CityZip Mankato," to be used as an Invocable Apex Method in a Flow.

Again, flow seems to effectively "`[0]`" invocable methods' return values, so return a `List` of whatever you actually want to pass back to the flow.

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
                spaceInJSONKey.matcher(response.getBody()).replaceAll('_'),
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
        System.assertEquals('MN',returnedCZ.state_abbreviation);
        System.assertEquals('Moorhead',returnedCZ.place_name);
        System.assert(returnedCZ.places[2].post_code.startsWith('5656'));
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
System.AssertException: Assertion Failed: CityZip:[country=United States, country_abbreviation=US, place_name=Mankato, places=(Place:[latitude=44.1538, longitude=-93.996, place_name=Mankato, post_code=56001], Place:[latitude=44.056, longitude=-94.0698, place_name=Mankato, post_code=56002], Place:[latitude=44.2172, longitude=-94.0942, place_name=Mankato, post_code=56003], Place:[latitude=44.056, longitude=-94.0698, place_name=Mankato, post_code=56006]), state=Minnesota, state_abbreviation=MN]
```

If you get this error, you need to enable making callouts to https://api.zippopotam.us in your org's [Remote Site Settings](https://login.salesforce.com/one/one.app#/setup/SecurityRemoteProxy/home) _(note that you can chop off `/us/mn/mankato`)_:

```
Line: 14, Column: 1
System.CalloutException: Unauthorized endpoint, please check Setup->Security->Remote site settings. endpoint = https://api.zippopotam.us/us/mn/mankato
```
