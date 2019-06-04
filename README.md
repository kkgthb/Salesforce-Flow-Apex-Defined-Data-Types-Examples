# Salesforce Flow Apex-Defined Data Types (code examples)

Contained within is a bit more Apex code to support the blog.

This is to help Salesforce Developers play around with Apex-Defined Data Types for Salesforce Flow, as used for calling out to HTTP APIs, in a bit more depth than is presented at [Tutorial: Flow Apex-Defined Data Types for Salesforce Admins](https://katiekodes.com/flow-apex-defined-data-types/) on  [Katie Kodes](https://katiekodes.com).

Many thanks to [this public API directory](https://github.com/public-apis/public-apis) for helping me find the examples.

## HTTPMockFactory class

You'll need this class for all 3 examples

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

## Indian City generator

Return a random Indian city thanks to https://indian-cities-api-nocbegfhqg.now.sh/

### IndianCity class

```java
public class IndianCity {
    @AuraEnabled @InvocableVariable public String city;
    @AuraEnabled @InvocableVariable public String state;
    @AuraEnabled @InvocableVariable public String district;
}
```

### IndianCityGenerator

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

### 
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
