# Polish REGON Internet Database BIR1

PHP bindings for the BIR1 (Baza Internetowa REGON 1) API (https://wyszukiwarkaregon.stat.gov.pl/appBIR/index.aspx).

[API Documentation](https://goo.gl/zxBf2o)

[![Latest Stable Version](https://poser.pugx.org/freshmindpl/wyszukiwarkaregon/v/stable)](https://packagist.org/packages/freshmindpl/wyszukiwarkaregon)
[![Build Status](https://travis-ci.org/freshmindpl/wyszukiwarkaregon.svg?branch=master)](https://travis-ci.org/freshmindpl/wyszukiwarkaregon)
[![Code Climate](https://codeclimate.com/github/freshmindpl/wyszukiwarkaregon/badges/gpa.svg)](https://codeclimate.com/github/freshmindpl/wyszukiwarkaregon)
[![Test Coverage](https://codeclimate.com/github/freshmindpl/wyszukiwarkaregon/badges/coverage.svg)](https://codeclimate.com/github/freshmindpl/wyszukiwarkaregon/coverage)

## Installation

The API client can be installed via [Composer](https://github.com/composer/composer).

In your composer.json file:

```js
{
    "require": {
        "freshmindpl/wyszukiwarkaregon": "~3.0"
    }
}
```

Once the composer.json file is created you can run `composer install` for the initial package install and `composer update` to update to the latest version of the API client.

## Basic Usage

Remember to include the Composer autoloader in your application:

```php
<?php
require_once 'vendor/autoload.php';

// Application code...
?>
```

Configure your access credentials when creating a client:

```php
<?php
use WyszukiwarkaRegon\Client;
use WyszukiwarkaRegon\Exception\RegonException;
use WyszukiwarkaRegon\Exception\SearchException;

$client = new Client([
   'key' => 'aaaabbbbccccdddd' //Optional api key - required for full reports,
   'session' => 'abcdefghijklmnopqrstuvwxyz' //Session id if already logged in
]);
?>
```

### Local Testing

Run `phpunit` from the project root to start all tests.

### Examples

#### Login

```php
<?php
// Login and obtain session id (sid)
try {
    $session_id = $client->login();
} catch (RegonException $e) {
    echo "There was an error.\n";
}

if(empty($session_id)) {
    // Empty session means that api key is invalid
    
    throw new \Exception('Invalid api key');
}

```

#### Logout

```php
<?php
// Login and obtain session id (sid)
try {
    $client->login();
} catch (RegonException $e) {
    echo "There was an error.\n";
}

....

// Invalidate current session
$client->logout();

```

#### GetValue

See [API Documentation](https://goo.gl/zxBf2o) for list of available params (section 2.5)
```php
<?php

try {
    $value = $client->getValue('StatusSesji');
} catch (RegonException $e) {
    echo "There was an error.\n";
}

?>
```

#### Search

```php
<?php

$params = [
    'Regon' => 142396858, // 9 or 14 digits
    'Krs' => null, // 10 digits
    'Nip' => null, // 10 digits
    'Regony9zn' => null, // Multiple 9 digits Regon's seperated by any non digit char (max 100)
    'Regony14zn' => null, // Multiple 14 digits Regon's seperated by any non digit char (max 100)
    'Krsy' => null, // Multiple 10 digits Krs seperated by any non digit char (max 100)
    'Nipy' => null, // Multiple 10 digits Nip seperated by any non digit char (max 100)
];

try {
    $data = $client->search($params);
    
} catch (SearchException $e) {
    switch($e->getCode()) {
        case GetValue::SEARCH_ERROR_CAPTCHA: //Captcha resolve needed
            // Some code
            break;
        case GetValue::SEARCH_ERROR_INVALIDARGUMENT: //Wrong search params
            // Some code
            break;
        case GetValue::SEARCH_ERROR_NOTFOUND: //Empty result - no data found matching search params
            // Some code
            break;
        case GetValue::SEARCH_ERROR_SESSION: //Wrong session id or expired session
            // Some code
            break;
    }
} catch (RegonException $e) {
    echo "There was an error.\n";
}
```

#### Reports

See [API Documentation](https://goo.gl/zxBf2o) for list of available reports (section 2.6)
```php
<?php

$regon = '1234567890';

try {
    $report = $client->report($regon, 'PublDaneRaportFizycznaOsoba'); 
    
} catch (SearchException $e) {
    switch($e->getCode()) {
        case GetValue::SEARCH_ERROR_CAPTCHA: //Captcha resolve needed
            // Some code
            break;
        case GetValue::SEARCH_ERROR_INVALIDARGUMENT: //Wrong search params
            // Some code
            break;
        case GetValue::SEARCH_ERROR_NOTFOUND: //Empty result - no data found matching search params
            // Some code
            break;
        case GetValue::SEARCH_ERROR_NOTAUTHORIZED: //Not authorized for this raport
            // Some code
            break;
        case GetValue::SEARCH_ERROR_SESSION: //Wrong session id or expired session
            // Some code
            break;
    }
} catch (RegonException $e) {
    echo "There was an error.\n";
}
```

### Full example

This is a working example on how to use GUS client to query for data

```php
<?php

$client = new Client([
    'key' => YOUR_API_KEY
]);

//Enable sandbox mode for development environment
if (defined('DEVELOPMENT') && DEVELOPMENT) {
    $client->sandbox();
}

//Check if we have saved session id
$session_id = $memcache->get('gus_session_id');

if (!$session_id) {
    try {
        $session_id = $client->login();
    } catch (RegonException $e) {
        echo "There was an error.\n";
        exit;
    }
    
    //Save session_id for later use
    $memcache->save('gus_session_id', $session_id); 
} else {

    //Set current session
    $client->setSession($session_id);
}

//Try to get data
try {
    
    //Get basic data
    $data = $client->search(['Nip' => '1234567890']);
    
    //Get full comapny report
    switch($data[0]['Typ']) {
        case 'P':
        case 'LP':
            $full = $client->report($data[0]['Regon'], 'PublDaneRaportPrawna');
        break;
    
        case 'F':
        case 'LF':
            $full = $client->report($data[0]['Regon'], 'PublDaneRaportDzialalnoscFizycznejCeidg');
        break;
    }
} catch (SearchException $e) {
    switch($e->getCode()) {
        case GetValue::SEARCH_ERROR_CAPTCHA: //Captcha resolve needed
            // You need to get catpcha and show it to the user
            break;
        case GetValue::SEARCH_ERROR_INVALIDARGUMENT: //Wrong search params
            // Invalid argument passed to search/report method
            break;
        case GetValue::SEARCH_ERROR_NOTFOUND: //Empty result - no data found matching search params
            // No records where found
            $data = null;
            $full = null;
            break;
        case GetValue::SEARCH_ERROR_NOTAUTHORIZED: //Not authorized for this raport
            // You are not authorized to generate this report
            break;
        case GetValue::SEARCH_ERROR_SESSION: //Wrong session id or expired session
            // Your session has expired - You need to login again
            break;
    }
} catch (RegonException $e) {
    echo "There was an error.\n";
    exit;
}

```

## Appendix

### Curl

curl --header "Content-Type: application/soap+xml; charset=utf-8" --data @soapReq https://wyszukiwarkaregontest.stat.gov.pl/wsBIR/UslugaBIRzewnPubl.svc -v

Body of soapReq file:
<?xml version="1.0" encoding="UTF-8"?>
<env:Envelope 
    xmlns:env="http://www.w3.org/2003/05/soap-envelope" 
    xmlns:ns1="http://CIS/BIR/PUBL/2014/07" 
    xmlns:ns2="http://www.w3.org/2005/08/addressing">
    <env:Header>
        <ns2:To>https://wyszukiwarkaregontest.stat.gov.pl/wsBIR/UslugaBIRzewnPubl.svc</ns2:To>
        <ns2:Action>http://CIS/BIR/PUBL/2014/07/IUslugaBIRzewnPubl/Zaloguj</ns2:Action>
    </env:Header>
    <env:Body>
        <ns1:Zaloguj>
            <ns1:pKluczUzytkownika>abcde12345abcde12345</ns1:pKluczUzytkownika>
        </ns1:Zaloguj>
    </env:Body>
</env:Envelope>

Response:
--uuid:b4f12732-43ef-4722-b1fe-dc75cbde0262+id=14980
Content-ID: <http://tempuri.org/0>
Content-Transfer-Encoding: 8bit
Content-Type: application/xop+xml;charset=utf-8;type="application/soap+xml"
<s:Envelope 
    xmlns:s="http://www.w3.org/2003/05/soap-envelope" 
    xmlns:a="http://www.w3.org/2005/08/addressing">
    <s:Header>
        <a:Action s:mustUnderstand="1">http://CIS/BIR/PUBL/2014/07/IUslugaBIRzewnPubl/ZalogujResponse</a:Action>
    </s:Header>
    <s:Body>
        <ZalogujResponse 
            xmlns="http://CIS/BIR/PUBL/2014/07">
            <ZalogujResult>zkuhtx73vb6mdv6tmf6n</ZalogujResult>
        </ZalogujResponse>
    </s:Body>
</s:Envelope>
--uuid:b4f12732-43ef-4722-b1fe-dc75cbde0262+id=14980--

### SoapUI

Create SOAP Project, add initial WSDL "https://wyszukiwarkaregontest.stat.gov.pl/wsBIR/wsdl/UslugaBIRzewnPubl.xsd" -> On Zaloguj/"Request 1" paste SOAP request from curl section and run. 

## License

MIT license. See the [LICENSE](LICENSE) file for more details.
