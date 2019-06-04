# Salesforce Flow Apex-Defined Data Types (code examples)

Contained within is a bit more Apex code to support my blog [Katie Kodes](https://katiekodes.com).

This is to help Salesforce Developers play around with Apex-Defined Data Types for Salesforce Flow, as used for calling out to HTTP APIs, in a bit more depth than is presented at [Tutorial: Flow Apex-Defined Data Types for Salesforce Admins](https://katiekodes.com/flow-apex-defined-data-types/).

Many thanks to [this public API directory](https://github.com/public-apis/public-apis) for helping me find the 3 examples _(YesNo, Shouting As A Service, and Indian Cities)_.

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

---

## YesNo

### YesNo class

```java
public class YesNo {
    @AuraEnabled @InvocableVariable public String answer;
    @AuraEnabled @InvocableVariable public Boolean forced;
    @AuraEnabled @InvocableVariable public String image;
}
```

### YesNoGenerator class

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

---

## SHOUTING AS A SERVICE

### AllCaps class

```java
public class AllCaps {
    @AuraEnabled @InvocableVariable public String input;
    @AuraEnabled @InvocableVariable public String output;
}
```

### AllCapsGenerator class

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

---

## Indian Cities

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
