# VEGAPI

VEGAPI is a hosted API for developers of business applications, handling accounting and taxes for every document like invoices, payments or bank deposits, that you post to it.  

Using VEGAPI you can enhance your business application by providing accounting and tax functionalities to your customers.  

Integration with VEGAPI is simple: your application talks to VEGAPI via HTTPS transactions, sending or receiving JSON objects and using familiar HTTP headers and return codes.  

The initial version of the API implements the following types of resources:

* **companies** : any organisation using your application to manage its business - basically a Company is your customer
* **documents** : any document - invoice, order, delivery note, expense report... - that a Company issues or receives, related to its business
* **payments** : any payment made or received by a Company related to a document
* **cash** : a set of resources that allow your application to manage a Company's cash accounts
* **entities** : any 3rd party with whom a Company has business - customers, suppliers, employees
* **items** : any product or service sold or purchased by a Company
* **accounting** : a set of resources that allow your application to review the accounting process of a Company, generate manual accounting entries, prepare acounting reports and handle other actions like closing a fiscal year
* **vat** : a set of resources that allow your application to generate and review VAT returns for a Company
* **drafts** : resources (of any type) that are being created / updated by your application but are not yet ready to be committed - basically a work area where your app can store "work in progess"  
* **users** and **profiles**: a set of resources that allow your application to control who is entitled to do what on each resource type  
* **settings** : a set of resources that allow your application to drive the financial, inventory, accounting and tax processes of VEGAPI
* **system** : a set of resources that facilitate administration and operation of VEGAPI itself.




-----
## Overview

To work with VEGAPI your application must be able to:

* support the HTTPS protocol and the Basic authentication method - see [Security](#overview/security)
* build URL to identify the resource you want to manipulate - see [Resource identification](#overview/resource-identification)
* handle JSON objects in the HTTP message body - see [Resource representation](#overview/resource-representation)
* send HTTP transactions with POST, GET, PUT and DELETE methods - see [HTTP methods](#overview/http-methods)
* set HTTP request headers and process HTTP response headers - see [HTTP headers](#overview/http-headers)
* process HTTP return codes - see [HTTP return codes](#overview/http-return-codes)
* set and process Range headers to support paged requests - see [Paged requests](#overview/paged-requests).

<br/>
For each document, payment or cash-transaction POSTed to it, VEGAPI will generate the required accounting entries and, on request, will create several inventory, tax and accounting reports and will perform other actions like closing a fiscal year - see [Accounting and tax](#overview/accounting-and-tax).  



### Security

To ensure security and confidentiality of data, the API forces the exclusive use of HTTPS through the header Strict-Transport-Security and uses the BASIC method for client application authentication - you can use the [signup page](https://signup.vegapi.org) to obtain an identifier and an authentication key for your application.


### Resource identification

VEGAPI resources are identified by an absolute URL in the format `https://{hostname}/{companyId}/{resourceId}` where {hostname} identifies a VEGAPI instance linked to a particular application, {companyId} identifies a particular business customer using this application and {resourceId} identifies a specific resource belonging to this business customer. 

Within HTTP transactions, VEGAPI resources can be identified by a relative URL in the format `/{companyId}/{resourceId}` with the HTTP Host Header identifying the VEGAPI instance - `{hostname}`. 

A VEGAPI resource URL can include an URL fragment (the part of the URL after the `#`symbol) which will be interpreted as the key of a key/value pair in the JSON representation of the resource.  


### Resource representation

VEGAPI uses the [JSON format](http://www.json.org) in the representation of resources. These representations can optionally be extended with new attributes defined by your application and, to avoid clashes, names of predefined VEGAPI attributes always start with a `_` (underscore).

Your application requests must include a JSON object in the request body, with a `_data` property containing the resource attributes:

```
{
	"_data": {                            // a JSON object
		...
		"_date": "20140723",              // vDate is predefined by VEGAPI
		...
		"partners": [                     // partners is defined by your app
			{"Jane Smith": 25000.00},
			{"John Smith": 25000.00}
		],
		...
	}
}
```


VEGAPI success responses (HTTP status codes 2xx) include in the response body a JSON object with several values, two of which always exist: `_data` (which can be a JSON object for individual resources or a JSON array for collections of resources) and `_links` (a JSON object with links to related resources). Other values like `_id`, `_status` or `_lastModifiedDate` can also exist:

```
{
	"_id": "/5v4080jn",
	"_data": {                             // a JSON object
		"_name": "My Company",             // _name is predefined by VEGAPI
		...,
		"partners": [                      // partners is defined by your app
			{"Jane Smith": 25000.00},
			{"John Smith": 25000.00}
		]
	},
    "_status": "active",                    // a predefined JSON string
    "_links": {								// a JSON object
		"_self": "/5v4080jn",				// _self is predefined by VEGAPI
		...
	}
}

// or
	
{
	"_data": [                             // a JSON array
		{
			"_name": "My Company", 
			"_id": "/5v4080jn"
		},
		{
			"_name": "Your Company", 
			"_id": "/wrgj410r"
		},
		{
			"_name": "Another Company", 
			"_id": "/xv2qk6dv"
		}
	],
	"_links": {								// a JSON object
		"_self": "/",						
		...
	}
}

```


VEGAPI error responses (HTTP status codes 4xx or 5xx) always include in the response body a JSON object detailing the error:

```
{
	"_error": {								// a JSON object
		"_code": "404",
		"_message": "The requested resource was not found"
	}
}
```


VEGAPI resource attributes are always formatted as JSON values: string, number, true, false, null, object and array. Additionally:

* dates are represented as strings in [ISO 8601 format](https://en.wikipedia.org/wiki/ISO_8601)
* country codes are represented as strings in [ISO 3166-1 alpha-2 format](https://en.wikipedia.org/wiki/ISO_3166-1_alpha-2)
* currency codes are represented as strings in [ISO 4217 format](https://en.wikipedia.org/wiki/ISO_4217)
* quantities and amounts that require a unit are represented by a JSON object with one or more key-value pairs, where the key contains the unit name and the value is the amount in that unit:
	* `"_quantity": {"Pieces": 5}` - represents a quantity of 5 pieces
	* `"_quantity": {"Boxes": 2}` - represents a quantity of 2 boxes
	* `"_amount": {"EUR": 640.28}` - represents an amount of 640.28 Euro.



### HTTP methods

Your application should support the following HTTP methods:

* GET : to request a resource or a collection of resources via its URL

* POST : to request the creation of a new resource; the new URL is defined by the API and returned in the Location header of the response

* PUT : to request the full update of a resource, using the representation included in the request body 

* DELETE : to request removal of the resource identified by the URL - see [Resource states](#overview/resource-states)



### HTTP headers

The following headers are mandatory in requests to VEGAPI and, if missing, trigger a 400 or 401 response from the API:

Header | Description
-------------------- | -------------------------------------------------------
`Host: {appId}.hostname:port` | to identify the API instance (linked to a specific application)
`Authorization: BASIC XXXXXXXXXXXXXX` | to authenticate the client application initially or upon receiving a 401 response and where XXXXXXXXXXX is the Base64 encoding of {appId}:{authKey}
`If-Match: XXXXXXXXXXXXXXXX` | to check the resource version (only mandatory for PUT or DELETE requests)


The following headers are optional in requests to VEGAPI:

Header | Description
-------------------- | -------------------------------------------------------
`Accept: application/json` | the media type the client application wants to receive - currently only JSON is supported
`Accept-Charset: UTF-8` | the media charset the client application wants to receive - must be UTF-8
`If-Modified-Since: date` | to minimize network traffic; triggers a 304 Not Modified response if the resource has not been modified since the indicated `date`
`Content-Type: application/json; charset=utf-8` | to communicate the media type of the request body for POST and PUT - currently only application/json with UTF8 is supported 
`Content-Length: NNNN` | to communicate the size of the request body for POST and PUT
`Range: items=n-m` | to request a specific range of items in a GET for a collection of resources - m items starting with item n (first item is zero)


The following headers are always used in VEGAPI responses:

Header | Description
-------------------- | -------------------------------------------------------
`Date: Fri, 19 Sep 2014 17:12:31 GMT` | the date and time the response was generated
`Strict-Transport-Security: max-age=31536000; IncludeSubdomains` | to force exclusive use of HTTPS
`Content-Type: application/json; charset=utf-8` | to communicate the media type of the response body - currently only application/json with UTF-8 is supported
`Last-Modified: date` | last known resource modification date
`ETag: XXXXXXXXXXXXXX` | ETag of the current version of the resource (only in GET and PUT responses)


The following headers may be used in VEGAPI responses:

Header | Description
-------------------- | -------------------------------------------------------
`WWW-Authenticate: BASIC realm={appID}` | request for the client application to authenticate itself to access VEGAPI instance {appId}
`Location: URL` | absolut URL of the new resource created as a result of a successful POST
`Accept-Ranges: items` | used by the API to indicate support for range requests in this resource - usually collection resources
`Content-Range: items n-m/p` | the API response contains `m` items starting with item `n`, with `p` being the total number of items in the collection
`Content-Length: NNNN` | to communicate the size of the response body
`Allow: GET, PUT, DELETE` | to communicate the list of HTTP methods supported for this resource



### HTTP return codes

The following HTTP return codes may be used by the API:

Header | Description
-------------------- | -------------------------------------------------------
`200 OK` | the request was successful and the response body contains the result
`201 Created` | the new resource was created and the Location header contains its URL
`202 Accepted` | the request has been accepted but its processing has not been completed
`204 No Content` | the request was successful, but the server is not returning any content in the response body
`206 Partial Content` | this response is part of the overall response, as indicated in the Range header
`304 Not Modified` | the resource was not modified since the version indicated in the request headers If-Modified-Since or If-Match
`400 Bad Request` | the request format is wrong
`401 Unauthorized` | the client application needs to authenticate itself; a WWW-Authenticate header is include in this response
`403 Forbidden` | the client application is not authorized to access the resource indicated
`404 Not Found` | the resource indicated does not exist
`405 Method Not Allowed` | this resource does not support the method indicated in the request
`406 Not Acceptable` | the API does not support the requested content type - only application/json is currently supported
`409 Conflict` | the request cannot be satisfied due to a conflict with the resource state
`410 Gone` | the resource has been deleted and is in the inactive state
`411 Length Required` | the Length header is missing in the request 
`412 Precondition Failed` | the resource ETag does not match the ETag indicated in the request
`413 Request Entity Too Large` | the request entity is larger than the API is able to process
`414 Request URL Too Long` | the request URL is larger than the API is able to process
`415 Unsupported Media Type` | the request entity is in a format not supported by the API
`418 Request Range Not Satisfiable` | the Range indicated in the request is invalid or impossible to satisfy
`428 Precondition Required` | this method requires an If-Match request header to prevent Lost Update problems
`429 Too Many Requests` | this application has sent too many requests in a short amount of time and should slowdown
`500 Internal Server Error` | an error has ocurred in the API server
`501 Not Implemented` | the request method is not implemented
`503 Service Unavailable` | the API is currently unavailable - please try later


### Paged requests

**ToDo**



### Resource states

VEGAPI resources can exist in several states represented by the value of the predefined string attribute `_status`. The following values are possible and cause specific API behaviors:

* **"empty"** - the status of a resource created by a POST request with an empty JSON object `{}` in its body  - see [Resource Integrity](#overview/resource-integrity). Empty resources:
	* are not included in responses to GET requests for resource collections, except when `?status=empty` is included in the query part of the URL
	* respond to individual GET, PUT and DELETE requests
	* are not processed by the API
   
* **"active"** - the (default) status of a resource created by a successful POST request with a valid `_data` attribute in the request body. Active resources:
	* are included in responses to GET requests for resource collections
	* respond to individual GET requests
	* are processed by the API

* **"deleted"** - the status of a resouce after a successful DELETE request. Deleted resources:
	* are not included in responses to GET requests for resource collections, except when `?status=deleted` is included in the query part of the URL
	* do not respond to individual GET requests (the API returns `410 Gone`)
	* are not processed by the API
	



### Resource integrity

To help preserve resource integrity, VEGAPI uses 2 mechanisms:

* To create a new resource your application can send a POST request with an empty JSON object `{}` in its body and, after receiving a 201 response, issue a PUT request to the newly created URL with the new resource representation; in the absence of a response to the POST, your application can simply resend it;
* In each PUT or DELETE request, your application must include a `If-Match` header with the known resource ETag; if this value doesn't match the current resource ETag, the request fails with a 412 Precondition Failed response.



### Accounting and tax

When your application creates or updates any resource, VEGAPI validates the request for format and content and, if correct, stores it in the database and sends back a 201 Created or a 200 OK response code. If applicable to the specific resource (namely documents, payments and cash-transactions), VEGAPI will then asynschronously generate the relevant financial and tax accounting entries based on the type of resource, the type of 3rd party entity, the type of items bought / sold (for invoices) or the bank account used (for payments or cash-transactions).   

To create these accounting entries, VEGAPI will also use several `/settings`, with values defined by your application, for things like ledger account numbers or ledger journal identification.  

For sales of inventory goods, VEGAPI will also perform an inventory check to calculate the correct cost of goods sold and to create the relevant accounting entries (currently VEGAPI supports FIFO and Weighted Average costing policies).  

If VEGAPI cannot successfully execute the full accounting process, no entries will be generated and an error will be posted to `/accounting-errors`;  the resource will still be retained, waiting for further actions to correct the error(s).  

Resources with accounting errors are reprocessed by VEGAPI whenever:
* the resource is successfully UPDATEd
* your application requests creation of a new `acounting-errors` report (for example, after creating a new setting)
* your application requests execution of another accounting process, like closing a fiscal year or generating a Balance Sheet report.  

VEGAPI provides your application with a number of additional endpoints to support the accounting process:
* `/accounting-errors` to provide the list of resources with accounting errors
* `/accounting-batches` to allow your application to directly generate batches of accounting entries
* `/account-statements`, `/balance-sheets`, `/income-statements` and `/cashflow-statements` to request generation of these accounting reports
*  `/fiscal-year-ends` to request the closing of one fiscal year and the transfer of accounting balances to the following one.  


If your application requires it, VEGAPI can support inventory lots (for tracking things like sizes / colours, expiry dates or individual serial numbers) via fragment strings in `/products` URLs (the last part of a URL after a `#` symbol).  

Tax support is provided by VEGAPI through an endpoint `vat-returns`, allowing your application to request generation of a new VAT-return for a given period, retrieve existing VAT-returns and, if absolutely needed, delete a VAT-return.  




-----
## API



### Companies

A Company, identified by the URL `/{companyId}`, is a legal entity using VEGAPI through your application for tax and accounting purposes; basically, a Company is your customer. 

The root endpoint `/` represents the list of all Companies registered in that API instance.  

A Company has the following pre-defined attributes:  

Name | Format | Description
---- | ------ | -----------
`_id` | string | A resource identifier set by VEGAPI (read only)
`_data` | object | Company attributes set by your application
`_data._name` | string | A secondary key used for queries and in lists
`_data._description` | string | Company full name
`_data._addresses` | object | List of company addresses
`_data._addresses._main` | string | Main company address
`_data._country` | string | Country of tax residence, in [ISO 3166-1 alpha-2 format](https://en.wikipedia.org/wiki/ISO_3166-1_alpha-2)
`_data._currency` | string | Currency for accounting and VAT purposes in [ISO 4217 format](https://en.wikipedia.org/wiki/ISO_4217)
`_data._taxNumber` | string | Tax identifier of Company
`_data._earliestVatDate` | string | Earliest date allowed for tax related documents to be processed, in [ISO 8601 format](https://en.wikipedia.org/wiki/ISO_8601)
`_data._earliestAccountingDate` | string | Earliest date allowed for any documents to be processed, in [ISO 8601 format](https://en.wikipedia.org/wiki/ISO_8601)
`_data._earliestInventoryDate` | string | Earliest date allowed for documents affecting inventory to be processed, in [ISO 8601 format](https://en.wikipedia.org/wiki/ISO_8601)
`_lastModifiedDate` | string | Server date of the last change to this resource, in [ISO 8601 format](https://en.wikipedia.org/wiki/ISO_8601) (read only)
`_status` | string | Resource status - see [Possible resource states](#overview/resource-states) (read only)
`_links` | object | Links to resources related to this one (read only)
`_links._self` | string | Link to this resource
`_links._documents` | string | Link to documents - invoices, cash sales, Db/Cr notes,...
`_links._payments` | string | Link to payments received/made
`_links._cashTransactions` | string | Link to cash-transactions
`_links._entities` | string | Link to entities
`_links._items` | string | Link to items - product / services
`_links._accountingBatches` | string | Link to batches of accounting entries
`_links._accountingErrors` | string | Link to list of accounting errors found in documents, payments, cash-transactions and accounting-batches
`_links._drafts` | string | Link to drafts of documents, payments, cash-transactions
`_links._vatReturns` | string | Link to vat returns / reports
`_links._fiscalYearEnds` | string | Link to fiscal-year-ends
`_links._accountStatements` | string | Link to Account Statement reports
`_links._balanceSheets` | string | Link to Balance Sheet reports
`_links._incomeStatements` | string | Link to Income Statement reports
`_links._cashflowStatement` | string | Link to Cashflow Statement reports
`_links._users` | string | Link to users
`_links._profiles` | string | Link to access profiles
`_links._settings` | string | Link to settings


<br/>  
`GET /?name={aCompanyName}` - requests a list of all Companies registered in this API instance, possibly filtered by `name` or `status`.

* `200 OK` - the response body contains a list of Companies, where _id contains a relative link to a Company

```
{
	"_data": [
		{
			"_name": "My Company",
			"_id": "/5v4080jn"
		},
		{
			"_name": "Your Company",
			"_id": "/wrgj410r"
		}
	],
	"_links": {
		"_self": "/"
	}
}
```
	
* `206 Partial Content` - The response body contains part of the response items as indicated in the Content-Range header.
	

* `401 Unauthorized` - The response body contains an error object

```
{
	"_error": {
		"_code": "401",
		"_message": "Your applicaton needs to be authenticated to access this API"
	}
}
```


<br/>
`POST /`- requests the creation of a new Company. The request body must contain a representation of the new resource or an empty JSON object - see [Resource states](#api/resource-states). 

```
{
	"_data": {
		"_name": "My Company",
		"_description": "My Company & Sons, Ltd",
		"_addresses": {
			"_main": "Some Street, 125, Room 213\nSome Town\n1234-056 Some Location\nPortugal"
		},
		"_country": "PT",
		"_currency": "EUR",
		"_taxNumber": "ABC123456789",
		"_earliestVatDate": "20140630",
        "_earliestInventoryDate": "20140101",
		"_earliestAccountingDate": "20140101"
	}
}
```

* `201 Created` - The response includes the headers Location (with the absolute URL for the new resource), Last-Modified and ETag. The response body contains the representation of the new resource

* `401 Unauthorized` - The response body contains an error object.



<br/>
`GET /{companyID}` - requests the representation of the Company identified by the URL. The request may include the header If-Modified-Since.

* `200 OK` - The response body contains the resource representation. The response includes the headers: Last-Modified and ETag.

```
{
    "_id": "/5v4080jn",
	"_data": {
		"_name": "My Company",
		"_description": "My Company & Sons, Ltd",
		"_addresses": {
			"_main": "Some Street, 125, Room 213\nSome Town\n1234-056 Some Location\nPortugal"
		},
		"_country": "PT",
		"_currency": "EUR",
		"_taxNumber": "ABC123456789",
		"_earliestVatDate": "20140630",
        "_earliestInventoryDate": "20140101",
		"_earliestAccountingDate": "20140101"
	},
    "_lastModifiedDate": "2014-07-2,
    "_status": "active",
	"_links": {
		"_self": "/5v4080jn",
		"_entities": "/5v4080jn/entities",
		"_items": "/5v4080jn/items",
		"_documents": "/5v4080jn/documents",
		"_payments": "/5v4080jn/payments",
		"_cashTransactions": "/5v4080jn/cash-transactions",
        "_drafts": "/5v4080jn/drafts",
        "_accountingBatches": "/5v4080jn/accounting-batches",
        "_accountingErrors": "/5v4080jn/accounting-errors",
        "_fiscalYearEnds": "/5v4080jn/fiscal-year-ends",
        "_accountStatements": "/5v4080jn/account-statements",
        "_balanceSheets": "/5v4080jn/balance-sheets",
        "_incomeStatements": "/5v4080jn/income-statements",
        "_cashflowStatements": "/5v4080jn/cashflow-statements",
		"_vatReturns": "/5v4080jn/vat-returns",
        "_users": "/5v4080jn/users",
        "_profiles": "/5v4080jn/profiles",
		"_settings": "/5v4080jn/settings"
	}
}
```


* `304 Not Modified` - The resource has not been modified since the date indicated in the request header.

* `404 Not Found` - The resource was not found. The response body contains additional error information

```
{
	"_error": {
		"_code": "404",
		"_message": "The requested resource was not found"
	}
}
```


* `410 Gone` - The resource has been deleted. The response body contains additional error information.

```
{
	"_error": {
		"_code": "410",
		"_message": "This resource has been deleted and is no longer available"
	}
}
```


<br/>
`PUT /{companyId}` - requests the replacement of the representation of the Company identified by the URL. The request body must contain the new representation of the resource, where the read-only properties may be omitted (they will be ignored).  The request must include the header: If-Match.

```
{
	"_data": {
		"_name": "My Company",
		"_description": "My Company & Sons, Ltd",
		"_addresses": {
			"_main": "Some Street, 125, Room 213\nSome Town\n1234-056 Some Location\nPortugal"
		},
		"_country": "PT",
		"_currency": "EUR",
		"_taxNumber": "ABC123456789",
		"_earliestVatDate": "20140630",
        "_earliestInventoryDate": "20140101",
		"_earliestAccountingDate": "20140101"
	}
}
```

* `200 OK` - The resource was updated. The response body contains the updated resource. The response includes the headers: Last-Modified and ETag.

* `404 Not Found` - The resource was not found. The response body contains additional error information.

* `409 Conflict` - The request was not executed as the resource's state would become inconsistent. The response body contains additional error information. The response includes the headers: Last-Modified and ETag.

```
{
	"_error": {
		"_code": "409",
		"_message": "You are trying to modify a read only property in this resource"
	}
}
```


* `412 Pre-condition Failed` - The request was not executed as the ETag supplied did not match the resources's current ETag. The response body contains additional error information. The response includes the headers: Last-Modified and ETag.

```
{
	"_error": {
		"_code": "412",
		"_message": "The resource you are trying to modify has been changed since you last read it"
	}
}
```


<br/>
`DELETE /{companyId}` - requests the removal of the Company identified by the URL. The request must include the header: If-Match.

* `204 No Content` - The resource was removed.

* `404 Not Found` - The resource was not found. The response body contains additional error information.

* `409 Conflict` - The request was not executed as the resource's state would become inconsistent. The response body contains additional error information. The response includes the headers: Last-Modified and ETag.

* `412 Pre-condition Failed` - The request was not executed as the ETag supplied did not match the resources's current ETag. The response body contains additional error information. The response includes the headers: Last-Modified and ETag.

-----


### Documents


A Document, identified by the URL `/{companyId}/documents/{documentId}`, represents a physical or electronic document (like an invoice, a credit note or a delivery note) exchanged between a [Company](#api/companies) and a third party [Entity](#api/entities) and related to a commercial transaction involving one or more [Products](#api/products) purchased or sold. **Note**: [Payments](#api/payments) made or received by Company are represented by a separate resource.

When your application creates or updates (or deletes) a Document, VEGAPI generates the required accounting entries, driven by `_documentType`, and `_itemType` - see [Overview](#overview/accounting-and-tax) for a description of VEGAPI accounting processes. The values of these properties must be defined in [Settings](#settings) for the accounting to be successful; otherwise the Document will be flagged as an [Accounting-error](#api/accounting-errors) (but will still be accepted and stored by VEGAPI).

* `documentType` defines the type of commercial document, with the most common document types predefined in VEGAPI: "CustomerInvoice", "SupplierInvoice", "CustomerCredit", "SupplierCredit", "OtherExpenses", "OtherRevenues". 

* `itemType` defines the type of item being puchased, bought, earned or expensed; several common item types are predefined in [Item-types](#settings/item-types), like "Materials", "Goods", "Salaries", "Supplies", "Repairs" and other.

A Document's `_externalNumber` may also be subject to certains requirements from the Tax Authorities, like being in sequence with no gaps.

A Document has the following pre-defined properties:

Name | Format | Description
---- | ------ | -----------
`_id` | string | Resource id set by VEGAPI (read only)
`_data` | object | Document attributes set by your application
`_data._name` | string |  A secondary key used in queries and in lists
`_data._description` | string | Document description
`_data._documentType` | string | The type of this document. It must exist as a key in [Settings/Document Types](#settings/document-types)
`_data._externalNumber` | string | A document number assigned by {CompanyId} and printed in the document; usually subject to specific requirements from Tax Authorities
`_data._date` | string | Date this document was issued
`_data._relatedDocuments` | array | Documents related to this one
`_data._relatedDocuments[i]._name` | string | Related document name
`_data._relatedDocuments[i]._id` | string | Link to related document
`_data._entity` | object | The entity related to this document
`_data._entity._name` | string | Entity's name
`_data._entity._id` | string | Link to entity
`_data._entity._description` | string | Entity's full name
`_data._entity._address` | string | Entity's address
`_data._entity._taxNumber` | string | Entity's tax identification
`_data._entity._countryCode` | string | Entity's country of tax residence
`_data._currency` | string | Currency of document
`_data._grossAmount` | object | Total gross amount of document in the document's currency and in the {companyId}'s currency - see [Quantities and amounts](#overview/resource-representation)
`_data._netAmount` | object | Total net amount of document in the document's currency and in the {companyId}'s currency
`_data._items` | array | Line items of document
`_data._items[i]._itemType` | string | Type of item. It must exist as a key in [Settings/Item Types](#settings/item-types)
`_data._items[i]._description` | string | Item description
`_data._items[i]._id` | string | Link to item; may include fragment in URL to identify a specific lot within a product (usually goods or materials)
`_data._items[i]._quantity` | object | Quantity of item
`_data._items[i]._grossAmount` | object | Item gross amount
`_data._items[i]._discountAmount` | object | Item discount amount
`_data._items[i]._netAmount` | object | Item net amount
`_data._items[i]._vatRate` | string | VAT rate code - see [Settings/VAT Rates](#settings/vat-rates)
`_data._items[i]._vatAmount` | object | VAT amount
`_lastModifiedDate` | string | Server date of the last change to resource (read only)
`_status` | string | Resource status (read only)
`_links` | record | Links to resources related to this one (read only)
`_links._self` | string | Link to this resource
`_links._company` | string | Link to this company
`_links._documents` | string | Link to documents sent to/received from the same Entity
`_links._payments` | string | Link to payments sent to/received from the same Entity


<br/>
`GET /{companyId}/documents[?name={aDocumentName}&entity={entityId}&item={itemId}&documentType={aDocumentType}&dates=(yyymmdd-yyymmdd)&status={aStatus}]` - Requests a list of all {companyId} Documents, optionally filtered by name / entity / item / document type / date / status. The request may include a Range header.

* `200 OK` - The response body contains a list of documents

```
{
	"_data": [
		{
			"_name": "EC 2014/135",
			"_id": "/5v4080jn/documents/av649h5z"
		},
		{
			"_name": "RC 2014/235",
			"_id": "/5v4080jn/documents/dw6492qa"
		},
		{
			"_name": "FC 2014/247",
			"_id": "/5v4080jn/documents/erjp90r1"
		},
		{
			"_name": "FF 1233-579/14",
			"_id": "/5v4080jn/documents/ero6j1rk"
		}
	],
	"_links": {
		"_self": "/5v4080jn/documents"
	}
}
```

* `206 Partial Content` - The response body contains part of the response as indicated in the Content-Range header.
	
* `401 Unauthorized` - The response body contains an error object

```
{
	"_error": {
		"_code": "401",
		"_message": "Your application needs to be authenticated to access this API"
	}
}
```


<br/>
`POST /{companyId}/documents` - Requests the creation of a new document. The request body must contain a representation of the new resource or an empty JSON object - see [Resource states](#api/resource-states).

```
{
	"_data": {
		"_name": "FC 2014/227",
		"_description": "",
		"_documentType": "CustomerInvoice",
		"_externalNumber": "FC - 2014/227",
		"_date": "2014-07-21",
		"_relatedDocuments": [
			{
				"_name": "RC 2014/235",
				"_id": "/5v4080jn/documents/dw6492qa"
			},
			{
				"_name": "EC 2014/135",
				"_id": "/5v4080jn/documents/av649h5z"
			}
		],
		"_entity": {
			"_id": "/5v4080jn/entities/4xrzy0v5",
			"_description": "A Customer, LLC",
			"_address": "A Street, 125, Room 213\nA Town\nA Location WX1234\nGreat Britain",
			"_countryCode" : "GB",
			"_taxNumber": "ABC123456789"
		},
		"_currency": "GBP",
		"_grossAmount": {"GBP": 6456.90, "EUR": 8910.52},
		"_netAmount": {"GBP": 5234.50, "EUR": 7223.61},
		"_items": [
			{
				"_itemType": "Goods",
				"_id": "/5v4080jn/items/ero6j1rk",
				"_description": "Woman Polo Shirt",
				"_quantity": {"Cartons": 10},
				"_grossAmount": {"GBP": 1500.00, "EUR": 2070.00},
				"_discountAmount": {"GBP": 75.00, "EUR": 103.50},
				"_netAmount": {"GBP": 1425.00, "EUR": 1966.50},
				"_vatRate": "Standard",
				"_vatAmount": {"GBP": 327.75, "EUR": 429.30}
			},
			{
				"_description": "Shipping",
				"_itemType": "Transports",
				"_id": "/5v4080jn/items/dn0j11vy",
				"_grossAmount": {"GBP": 15.00, "EUR": 20.70},
				"_netAmount": {"GBP": 15.00, "EUR": 20.70},
				"_vatRate": "Standard",
				"_vatAmount": {"GBP": 3.28, "EUR": 4.29}
			}
		]
	}
}

```

* `201 Created` - The response includes the headers Location (with the absolute URL for the new resource), Last-Modified and ETag. The response body contains a representation of the new resource

* `401 Unauthorized` - The response body contains an error object.


<br/>
`GET /{companyID}/documents/{documentId}` - Requests a specific document, identified by the URL. The request may include the header If-Modified-Since.

* `200 OK` - The response body contains the resource. The response includes the headers: Last-Modified and ETag.

```
{
    "_id": "/5v4080jn/documents/erjp90r1",
	"_data": {
		"_name": "FC 2014/227",
		"_description": "",
		"_documentType": "CustomerInvoice",
		"_externalNumber": "FC - 2014/227",
		"_date": "2014-07-21",
		"_relatedDocuments": [
			{
				"_name": "RC 2014/235",
				"_id": "/5v4080jn/documents/dw6492qa"
			},
			{
				"_name": "EC 2014/135",
				"_id": "/5v4080jn/documents/av649h5z"
			}
		],
		"_entity": {
			"_id": "/5v4080jn/entities/4xrzy0v5",
			"_description": "A Customer, LLC",
			"_address": "A Street, 125, Room 213\nA Town\nA Location WX1234\nGreat Britain",
			"_countryCode" : "GB",
			"_taxNumber": "ABC123456789"
		},
		"_currency": "GBP",
		"_grossAmount": {"GBP": 6456.90, "EUR": 8910.52},
		"_netAmount": {"GBP": 5234.50, "EUR": 7223.61},
		"_items": [
			{
				"_itemType": "Goods",
				"_id": "/5v4080jn/items/ero6j1rk",
				"_description": "Woman Polo Shirt",
				"_quantity": {"Cartons": 10},
				"_grossAmount": {"GBP": 1500.00, "EUR": 2070.00},
				"_discountAmount": {"GBP": 75.00, "EUR": 103.50},
				"_netAmount": {"GBP": 1425.00, "EUR": 1966.50},
				"_vatRate": "Standard",
				"_vatAmount": {"GBP": 327.75, "EUR": 429.30}
			},
			{
				"_description": "Shipping",
				"_itemType": "Transports",
				"_id": "/5v4080jn/items/dn0j11vy",
				"_grossAmount": {"GBP": 15.00, "EUR": 20.70},
				"_netAmount": {"GBP": 15.00, "EUR": 20.70},
				"_vatRate": "Standard",
				"_vatAmount": {"GBP": 3.28, "EUR": 4.29}
			}
		]
	},
,
    "_status": "active",
    "_lastModifiedDate": "2014-07-21T10:37:45Z",	
    "_links": {
		"_self": "/5v4080jn/documents/erjp90r1",
        "_company": "/5v4080jn",
		"_documents": "/5v4080jn/documents?entity=4xrzy0v5",
		"_payments": "/5v4080jn/payments/?entity=4xrzy0v5"
	}
}
```

* `304 Not Modified` - The resource has not been modified since the date indicated in the request header.

* `404 Not Found` - The resource was not found. The response body contains additional error information

```
{
	"_error": {
		"_code": "404",
		"_message": "The requested resource was not found"
	}
}
```

* `410 Gone` - The resource has been deleted. The response body contains additional error information

```
{
	"_error": {
		"_code": "410",
		"_message": "This resource has been deleted and is no longer available"
	}
}
```


<br/>
`PUT /{companyId}/documents/{documentId}` - Requests the replacement of the document identified by the URL. The request body must contain a representation of the new resource, where the read-only properties may be omitted (they will be ignored). The request must include the header: If-Match.

```
{
	"_data": {
		"_name": "FC 2014/227",
		"_description": "",
		"_documentType": "CustomerInvoice",
		"_externalNumber": "FC - 2014/227",
		"_date": "2014-07-21",
		"_relatedDocuments": [
			{
				"_name": "RC 2014/235",
				"_id": "/5v4080jn/documents/dw6492qa"
			},
			{
				"_name": "EC 2014/135",
				"_id": "/5v4080jn/documents/av649h5z"
			}
		],
		"_entity": {
			"_id": "/5v4080jn/entities/4xrzy0v5",
			"_description": "A Customer, LLC",
			"_address": "A Street, 125, Room 213\nA Town\nA Location WX1234\nGreat Britain",
			"_countryCode" : "GB",
			"_taxNumber": "ABC123456789"
		},
		"_currency": "GBP",
		"_grossAmount": {"GBP": 6456.90, "EUR": 8910.52},
		"_netAmount": {"GBP": 5234.50, "EUR": 7223.61},
		"_items": [
			{
				"_itemType": "Goods",
				"_id": "/5v4080jn/items/ero6j1rk",
				"_description": "Woman Polo Shirt",
				"_quantity": {"Cartons": 10},
				"_grossAmount": {"GBP": 1500.00, "EUR": 2070.00},
				"_discountAmount": {"GBP": 75.00, "EUR": 103.50},
				"_netAmount": {"GBP": 1425.00, "EUR": 1966.50},
				"_vatRate": "Standard",
				"_vatAmount": {"GBP": 327.75, "EUR": 429.30}
			},
			{
				"_description": "Shipping",
				"_itemType": "Transports",
				"_id": "/5v4080jn/items/dn0j11vy",
				"_grossAmount": {"GBP": 15.00, "EUR": 20.70},
				"_netAmount": {"GBP": 15.00, "EUR": 20.70},
				"_vatRate": "Standard",
				"_vatAmount": {"GBP": 3.28, "EUR": 4.29}
			}
		]
	}
}
```

* `200 OK` - The resource was updated. The response body contains the updated resource. The response includes the headers: Last-Modified and ETag.

* `404 Not Found` - The resource was not found. The response body contains additional error information.

* `409 Conflict` - The request was not executed as the resource's state would become inconsistent. The response body contains additional error information. The response includes the headers: Last-Modified and ETag

```
{
	"_error": {
		"_code": "409",
		"_message": "Total gross amount for document does not equal sum of gross amounts for items"
	}
}
```


* `412 Pre-condition Failed` - The request was not executed as the ETag supplied did not match the resources's current ETag. The response body contains additional error information. The response includes the headers: Last-Modified and ETag

```
{
	"_error": {
		"_code": "412",
		"_message": "The resource you are trying to modify has been changed since you last read it"
	}
}
```


<br/>
`DELETE /{companyId}/documents/{documentId}` - Requests the removal of the document identified by the URL - see [Resource states](#overview/resource-states). The request must include the header: If-Match.

* `204 No Content` - The resource was removed.

* `404 Not Found` - The resource was not found. The response body contains additional error information.

* `409 Conflict` - The request was not executed as the resource's state would become inconsistent. The response body contains additional error information. The response includes the headers: Last-Modified and ETag.

* `412 Pre-condition Failed` - The request was not executed as the ETag supplied did not match the resources's current ETag. The response body contains additional error information. The response includes the headers: Last-Modified and ETag.

-----

### Payments


A Payment, identified by the URL `/{companyId}payments/{paymentId}`, represents a payment made or received by a [Company](#api/companies) to or from a third-party [entity](#api/entities).

A Payment is always made through a [cash-account](#settings/cash-accounts).

A Payment can include a list of [payment-instruments](#settings/payment-instruments) used: cash, cheques, credit / debit cards, bank transfers.

A Payment can contain a list of [documents](#api/documents) to settle and respective amounts for each.

A Payment has the following pre-defined attributes:

Name | Format | Description
---- | ------ | -----------
`_id` | string | Resource id set by VEGAPI (read only)
`_data` | object | Payment attributes set by your application
`_data._name` | string | A secondary key used in queries and in lists
`_data._paymentType` | string | The type of payment; can be: "Sent" or "Received"
`_data._externalNumber` | string | A document number, unique within {companyId}
`_data._date` | string | Date this payment was made
`_data._entity` | object | The entity related to this payment
`_data._entity._name` | string | Entity's name
`_data._entity._id` | string | Link to entity
`_data._cashAccount` | object | The cash-account where this payment was made from /received into
`_data._cashAccount._name` | string | The cash-account's name
`_data._cashAccount._id` | string | Link to cash-account
`_data._amount` | object | Total amount of payment in the payment's currency and in the {companyId}'s currency - see [Resource representation](#overview/resource-representation)
`_data._instruments` | array | List of payment instruments that make up the payment
`_data._instruments[i]._name` | string | Type of payment instrument
`_data._instruments[i]._amount` | object | Amount in this payment instrument
`_data._applyTo` | array | List of documents settled by this payment
`_data._applyTo[i]._name` | string | Name of document settled 
`_data._applyTo[i]._id` | string | Link to document settled
`_data._applyTo[i]._amount` | object | Amount settled in this document
`_lastModifiedDate` | string | Server date of the last change to resource (read only)
`_status` | string | Resource status - see [Resource states](#overview/resource-states) (read only) 
`_links` | record | Links to resources related to this one (read only)
`_links._self` | string | Link to this resource
`_links._company` | string | Link to company owning this resource
`_links._documents` | string | Link to documents related to the same entity
`_links._payments` | string | Link to payments related to the same entity



<br/>
`GET /{companyId}/payments[?name={aPaymentName}&entity={entityId}&account={accountId}&date=(yyyymmdd-yyyymmdd)&status={aStatus}]` - Requests a list of all payments, optionally filtered by name or entity and/or account and/or date or status. The request may include a Range header.

* `200 OK` - The response body contains a list of documents

```
{
	"_data": [
		{
			"_name": "PC 2014/94",
			"_id": "/5v4080jn/payments/hd562rqs"
		},
		{
			"_name": "PF 2014/95",
			"_id": "/5v4080jn/payments/j6vnwz09"
		}
	],
	"_links": {
		"_self": "/5v4080jn/payments"
	}
}
```

* `206 Partial Content` - The response body contains part of the response as indicated in the Content-Range header.
	
* `401 Unauthorized` - The response body contains an error object

```
{
	"_error": {
		"_code": "401",
		"_message": "Your applicaton needs to be authenticated to access this API"
	}
}
```


<br/>
`POST /{companyId}/payments` - Requests the creation of a new payment. The request body must contain a representation of the new resource or an empty JSON object - see [Resource states](#api/resource-states).

```
{
	"_data": {
		"_name": "PC 2014/94",
		"_paymentType": "Received",
		"_externalNumber": "PC - 2014/94",
		"_date": "2014-07-21",
		"_entity": {
			"_name": "A Customer",
			"_id": "/5v4080jn/entities/4xrzy0v5"
			},
		"_cashAccount": {
			"_name": "Cashier HO",
			"_id": "/5v4080jn/accounts/hyd920tr"
			},
		"_amount": {"GBP": 6456.90, "EUR": 8910.52},
		"_instruments": [
			{
				"_name": "Cash",
				"_amount": {"GBP": 456.90}
			},
			{
				"_name": "Cheque",
				"_amount": {"GBP": 6000.00}
			}
		],
		"_applyTo": [
			{
				"_name": "FC 2014/227",
				"_id": "/5v4080jn/documents/erjp90r1",
				"_amount": {"GBP": 6456.90}
			}
		]
	}
}
```

* `201 Created` - The response includes the headers Location (with the absolute URL for the new resource), Last-Modified and ETag. The response body contains the representation of the new resource.

* `401 Unauthorized` - The response body contains an error object.



<br/>
`GET /{companyID}/payments/{paymentId}` - Requests a specific payment, identified by the URL. The request may include the header If-Modified-Since.

* `200 OK` - The response body contains the resource. The response includes the headers: Last-Modified and ETag.

```
{
	"_data": {
		"_id": "hd562rqs",
		"_name": "PC 2014/94",
		"_paymentType": "Received",
		"_externalNumber": "PC - 2014/94",
		"_date": "2014-07-21",
		"_entity": {
			"_name": "A Customer",
			"_id": "/5v4080jn/entities/4xrzy0v5"
			},
		"_cashAccount": {
			"_name": "Cashier HO",
			"_id": "/5v4080jn/accounts/hyd920tr"
			},
		"_amount": {"GBP": 6456.90, "EUR": 8910.52},
		"_instruments": [
			{
				"_name": "Cash",
				"_amount": {"GBP": 456.90}
			},
			{
				"_name": "Cheque",
				"_amount": {"GBP": 6000.00}
			}
		],
		"_applyTo": [
			{
				"_name": "FC 2014/227",
				"_id": "/5v4080jn/documents/erjp90r1",
				"_amount": {"GBP": 6456.90}
			}
		]	
    },
    "_status": "active",
    "_lastModifiedDate": "2014-07-21T10:37:45Z",
	"_links": {
		"_self": "/5v4080jn/payments/hd562rqs",
        "_company": "/5v4080jn",
		"_documents": "/5v4080jn/documents?entity=4xrzy0v5",
		"_payments": "/5v4080jn/payments?entity=4xrzy0v5"
	}
}
```

* `304 Not Modified` - The resource has not been modified since the date indicated in the request header.

* `401 Unauthorized` - The response body contains an error object.

* `404 Not Found` - The resource was not found. The response body contains additional error information

```
{
	"_error": {
		"_code": "404",
		"_message": "The requested resource was not found"
	}
}
```

* `410 Gone` - The resource has been deleted. The response body contains additional error information

```
{
	"_error": {
		"_code": "410",
		"_message": "This resource has been deleted and is no longer available"
	}
}
```


<br/>
`PUT /{companyId}/payments/{paymentId}` - Requests the replacement of the payment identified by the URL. The request body must contain a representation of the new resource, where the read-only properties may be omitted (they will be ignored).  The request must include the header: If-Match.

```
{
	"_data": {
		"_name": "PC 2014/94",
		"_paymentType": "Received",
		"_externalNumber": "PC - 2014/94",
		"_date": "2014-07-21",
		"_entity": {
			"_name": "A Customer",
			"_id": "/5v4080jn/entities/4xrzy0v5"
			},
		"_cashAccount": {
			"_name": "Cashier HO",
			"_id": "/5v4080jn/accounts/hyd920tr"
			},
		"_amount": {"GBP": 6456.90, "EUR": 8910.52},
		"_instruments": [
			{
				"_name": "Cash",
				"_amount": {"GBP": 456.90}
			},
			{
				"_name": "Cheque",
				"_amount": {"GBP": 6000.00}
			}
		],
		"_applyTo": [
			{
				"_name": "FC 2014/227",
				"_id": "/5v4080jn/documents/erjp90r1",
				"_amount": {"GBP": 6456.90}
			}
		]
	}
}
```

* `200 OK` - The resource was updated. The response body contains the updated resource. The response includes the headers: Last-Modified and ETag.

* `401 Unauthorized` - The response body contains an error object.

* `404 Not Found` - The resource was not found. The response body contains additional error information.

* `409 Conflict` - The request was not executed as the resource's state would become inconsistent. The response body contains additional error information. The response includes the headers: Last-Modified and ETag

```
{
	"_error": {
		"_code": "409",
		"_message": "You are trying to modify a payment from an already closed fiscal year"
}
```

* `412 Pre-condition Failed` - The request was not executed as the ETag supplied did not match the resources's current ETag. The response body contains additional error information. The response includes the headers: Last-Modified and ETag

```
{
	"_error": {
		"_code": "412",
		"_message": "The resource you are trying to modify has been changed since you last read it: 3voj529d"
	}
}
```



<br/>
`DELETE /{companyId}/payments/{paymentId}` - Requests the removal of the payment identified by the URL. The request must include the header: If-Match.

* `204 No Content` - The resource was removed.

* `404 Not Found` - The resource was not found. The response body contains additional error information.

* `409 Conflict` - The request was not executed as the resource's state would become inconsistent. The response body contains additional error information. The response includes the headers: Last-Modified and ETag.

* `412 Pre-condition Failed` - The request was not executed as the ETag supplied did not match the resources's current ETag. The response body contains additional error information. The response includes the headers: Last-Modified and ETag.



-----

### Cash-transactions


A Cash-transaction, identified by the URL `/{companyId}/cash-transactions/{transactionId}`, represents an operation that moves money between 2 cash accounts. Money in a Cash-transaction is represented by one or more [payment-instruments](#settings/payment-instruments) of several types (banknotes / coins, cheques, bank transfers...). 

Note: payments made into or from a Cash Account are represented by [Payments](#api/payments)).  

A Cash-transaction has the following pre-defined attributes:  

Name | Format | Description
---- | ------ | -----------
`_id` | string | Resource id set by VEGAPI (read only)
`_data` | object | Cash-transaction attributes set by your application
`_data._name` | string | A secondary key used for queries and in lists
`_data.description` | string | Description of transaction 
`_data._externalNumber` | string | A document number, unique within {companyId} and usually subject to specific requirements from Tax Authorities
`_data._date` | string | Date of this transaction
`_data._amount` | object | Amount of transaction
`_data._cashAccountFrom` | string | Name of source cash-account
`_data._cashAccountTo` | string | Name of destination cash-account
`_data._instruments` | array | List of payment-instruments included in this transaction
`_data._instruments[i]._name` | string | Type of payment-instrument - see [Payment-instruments](#settings/payment-instruments) 
`_data._instruments[i]._amount` | object | Amount in this payment-instrument
`_lastModifiedDate` | string | Server date of the last change to resource (read only)
`_status` | string | Resource status 
`_links` | record | Links to resources related to this one (read only)
`_links._self` | string | Link to this resource
`_links._company` | string | Link to company owning this resource
`_links._transactions` | string | Link to cash-transactions


<br/>
`GET /{companyId}/cash-transactions[?name={aTransactionName}&account={accountId}&date=(yyyymmdd-yyyymmdd)&status={aStatus}]` - Requests a list of all cash transactions made by {companyId}, optionally filtered by name or cash-account and/or dates or status. The request may include a Range header.

* `200 OK` - the response body contains a list of cash transactions

```
{
	"_data": [
		{
			"_name": "MT-2014/047",
			"_id": "/5v4080jn/cash-transactions/pv952aa1"
		},
		{
			"_name": "MT-2014/046",
			"_id": "/5v4080jn/cash-transactions/woxyzj08"
		}
	],
	"_links": {
		"_self": "/5v4080jn/cash-transactions"
	}
}
```
	
* `206 Partial Content` - The response body contains part of the response items, as indicated in the Content-Range header.

* `401 Unauthorized` - The response body contains an error object

```
{
	"_error": {
		"_code": "401",
		"_message": "Your applicaton needs to be authenticated to access this API"
	}
}
```


<br/>
`POST /{companyId/cash-transactions` - Requests the creation of a new cash transaction. The request body must contain a representation of the new resource or an empty JSON object - see [Resource states](#api/resource-states).

```
{
	"_data": {
		"_name": "MT-2014/047",
		"_description": "Cheque deposit 21 March 2014",
		"_externalNumber": "MT - 2014/047",
		"_date": "2014-03-21",
		"_cashAccountFrom":	"Cashier HO",
		"_cashAccountTo": "BPI-DO",
		"_instruments": [
			{
				"_name": "Cheque",
				"_amount": {"GBP": 6000.00}
			},
			{
				"_name": "Cash",
				"_amount": {"EUR": 2000.00}
			}
		],
		"_amount": {"EUR": 8871.14}
	}
}

```

* `201 Created` - The response includes the headers Location (with the absolute URL for the new resource), Last-Modified and ETag. The response body contains the representation of the new resource.

* `401 Unauthorized` - The response body contains an error object.



<br/>
`GET /{companyID}/cash-transactions/{transactionId}` - Requests a specific cash transaction, identified by the URL. The request may include the header If-Modified-Since.

* `200 OK` - The response body contains the resource. The response includes the headers: Last-Modified and ETag.

```
{
    "_id": "/5v4080jn/cash-transactions/pv952aa1",
	"_data": {
		"_name": "MT-2014/047",
		"_description": "Cheque deposit 21 March 2014",
		"_externalNumber": "MT - 2014/047",
		"_date": "2014-03-21",
		"_cashAccountFrom":	"Cashier HO",
		"_cashAccountTo": "BPI-DO",
		"_instruments": [
			{
				"_name": "Cheque",
				"_amount": {"GBP": 6000.00}
			},
			{
				"_name": "Cash",
				"_amount": {"EUR": 2000.00}
			}
		],
		"_amount": {"EUR": 8871.14}
	},
    "_status": "active",
    "_lastModifiedDate": "2014-07-21T10:37:45Z",
	"_links": {
		"_self": "/5v4080jn/cash-transactions/pv952aa1",
        "_company": "/5v4080jn"
		"_transactions": "/5v4080jn/cash-transactions"
	}
}
```

* `304 Not Modified` - The resource has not been modified since the date indicated in the request header.

* `404 Not Found` - The resource was not found. The response body contains additional error information.

```
{
	"_error": {
		"_code": "404",
		"_message": "The requested resource was not found: 3voj529d"
	}
}
```

* `410 Gone` - The resource has been deleted. The response body contains additional error information.

```
{
	"_error": {
		"_code": "410",
		"_message": "This resource has been deleted and is no longer available: 3voj529d"
	}
}
```


<br/>
`PUT /{companyId}/cash-transactions/{transactionId}` - Requests the replacement of the cash transaction identified by the URL. The request body must contain a representation of the new resource, where the read-only attributes may be omitted (they will be ignored). The request must include the header: If-Match.

```
{
	"_data": {
		"_name": "MT-2014/047",
		"_description": "Cheque deposit 21 March 2014",
		"_externalNumber": "MT - 2014/047",
		"_date": "2014-03-21",
		"_cashAccountFrom":	"Cashier HO",
		"_cashAccountTo": "BPI-DO",
		"_instruments": [
			{
				"_name": "Cheque",
				"_amount": {"GBP": 6000.00}
			},
			{
				"_name": "Cash",
				"_amount": {"EUR": 2000.00}
			}
		],
		"_amount": {"EUR": 8871.14}
	}
}
```

* `200 OK` - The resource was updated. The response body contains the updated resource. The response includes the headers: Last-Modified and ETag.

* `404 Not Found` - The resource was not found. The response body contains additional error information.

* `409 Conflict` - The request was not executed as the resource's state would become inconsistent. The response body contains additional error information. The response includes the headers: Last-Modified and ETag.

```
{
	"_error": {
		"_code": "409",
		"_message": "You are trying to modify a cash transaction already included in a tax submission"
	}
}
```

* `412 Pre-condition Failed` - The request was not executed as the ETag supplied did not match the resources's current ETag. The response body contains additional error information. The response includes the headers: Last-Modified and ETag.

```
{
	"_eeror": {
		"_code": "412",
		"_message": "The resource you are trying to modify has been changed since you last read it: 3voj529d"
	}
}
```



<br/>
`DELETE /{companyId}/cash-transactions/{transactionId}` - Requests the removal of the cash transaction identified by the URL. The request must include the header: If-Match.

* `204 No Content` - The resource was removed.

* `404 Not Found` - The resource was not found. The response body contains additional error information.

* `409 Conflict` - The request was not executed as the resource's state would become inconsistent. The response body contains additional error information. The response includes the headers: Last-Modified and ETag.

* `412 Pre-condition Failed` - The request was not executed as the ETag supplied did not match the resources's current ETag. The response body contains additional error information. The response includes the headers: Last-Modified and ETag.



-----

### Entities


An Entity, identified by the URL `/{companyId}/entities/{entityId}`, is any third party organisation or person with whom a [Company](#api/companies) does business - a customer, a supplier, an employee...  

Although its usage is not strictly needed, it can become convenient as a resource where your application can store fixed data about a customer or a supplier.  

An Entity has the following pre-defined attributes:


Name | Format | Description
---- | ------ | -----------
`_id` | string | Resource id set by VEGAPI (read only)
`_data._name` | string | A secondary key set by your application and used in queries and lists
`_data._description` | string | Entity full name
`_data._taxNumber` | string | Entity tax identification number
`_data._addresses` | object | List of company addresses, of which one must have the key `_main` (default address)
`_data._country` | string | Country of tax residence, in [ISO 3166-1 alpha-2 format](https://en.wikipedia.org/wiki/ISO_3166-1_alpha-2)
`_data._currency` | string | Default currency for this entity, to be used if a specific currency is not included in document representations; see [ISO 4217 format](https://en.wikipedia.org/wiki/ISO_4217)
`_data._tags` | array of strings | Free strings to tag this entity, defined and set by your application, mainly used to filter entity lists
`_lastModifiedDate` | string | Server date of the last change to resource, in [ISO 8601 format](https://en.wikipedia.org/wiki/ISO_8601) (read only)
`_status` | string | Resource status - see [Resource states](#overview/resource-states) (read only)
`_links` | record | Links to resources related to this one (read only)
`_links._self` | string | Link to this resource
`_links._company` | string | Link to company owning this resource
`_links._documents` | string | Link to documents sent to or received from this entity
`_links._payments` | string | Link to payments received from or made to this entity



<br/>
`GET /{companyId}/entities[?name={anEntityName}&tags=("www", "zzz"...)&status={aStatus}]` - Requests a list of all entities created by {companyId}, optionally filtered by name or tags or status. The request may include a Range header.

* `200 OK` - the response body contains a list of entities satisfying the query constraints

```
{
	"_data": [
		{
			"_name": "A Customer",
			"_id": "/5v4080jn/entities/4xrzy0v5"
		},
		{
			"_name": "A Supplier",
			"_id": "/5v4080jn/entities/ygo41pow"
		}
	],
	"_links": {
		"_self": "/entities"
	}
}
```
	
* `206 Partial Content` - The response body contains part of the response items as indicated in the Content-Range header.
	
* `401 Unauthorized` - The response body contains an error object

```
{
	"_error": {
		"_code": "401",
		"_message": "Your applicaton needs to be authenticated to access this API"
	}
}
```



<br/>
`POST /{companyId}/entities` - Requests the creation of a new entity. The request body must contain a representation of the new resource or an empty JSON object - see [Resource states](#api/resource-states).

```
{
	"_data": {
		"_name": "A Customer",
		"_description": "A Customer, LLC",
		"_country": "GB",
		"_taxNumber": "ABC123456789",
		"_currency": "GBP",
		"_tags": ["Grande Retalho"],
		"_addresses": {
			"_main": "A Street, 125, Room 213\nA Town\nA Location WX1234\nGreat Britain",
			"Warehouse": "Second Street, 250\nSome Town\nSome Location WX1234\nGreat Britain"
		}
	}
}
```

* `201 Created` - The response includes the headers Location (with the absolute URL for the new resource), Last-Modified and ETag. The response body contains a representation of the new resource.

* `401 Unauthorized` - The response body contains an error object.



<br/>
`GET /{companyID}/entities/{entityId}` - Requests the full representation of the entity identified by the URL. The request may include the header If-Modified-Since.

* `200 OK` - The response body contains the resource. The response includes the headers: Last-Modified and ETag.

```
{
    "_id": "/5v4080jn/entities/4xrzy0v5",
	"_data": {
		"_name": "A Customer",
		"_description": "A Customer, LLC",
		"_country": "GB",
		"_taxNumber": "ABC123456789",
		"_currency": "GBP",
		"_tags": ["Grande Retalho"],
		"_addresses": {
			"_main": "A Street, 125, Room 213\nA Town\nA Location WX1234\nGreat Britain",
			"Warehouse": "Second Street, 250\nSome Town\nSome Location WX1234\nGreat Britain"
		}
	},
    "_lastModifiedDate": "2014-07-21T10:37:45Z",
    "_status": "active",
	"_links": {
		"_self": "/5v4080jn/entities/4xrzy0v5",
        "_company": "/5v4080jn",
		"_documents": "/5v4080jn/documents?entity=4xrzy0v5",
		"_payments": "/5v4080jn/payments?entity=4xrzy0v5"
	}
}
```

* `304 Not Modified` - The resource has not been modified since the date indicated in the request header.

* `404 Not Found` - The resource was not found. The response body contains additional error information

```
{
	"_error": {
		"_code": "404",
		"_message": "The requested resource was not found: 3voj529d"
		}
}
```

* `410 Gone` - The resource has been deleted. The response body contains additional error information

```
{
	"_error": {
		"_code": "410",
		"_message": "This resource has been deleted and is no longer available: 3voj529d"
	}
}
```



<br/>
`PUT /{companyId}/entities/{entityId}` - Requests the replacement of the entity identified by the URL. The request body must contain a representation of the new resource, where the read-only properties may be omitted (they will be ignored). The request must include the header: If-Match.

```
{
	"_data": {
		"_name": "A Customer",
		"_description": "A Customer, LLC",
		"_country": "GB",
		"_taxNumber": "ABC123456789",
		"_currency": "GBP",
		"_tags": ["Grande Retalho"],
		"_addresses": {
			"_main": "A Street, 125, Room 213\nA Town\nA Location WX1234\nGreat Britain",
			"Warehouse": "Second Street, 250\nSome Town\nSome Location WX1234\nGreat Britain"
		}
	}
}
```

* `200 OK` - The resource was updated. The response body contains the updated resource. The response includes the headers: Last-Modified and ETag.

* `404 Not Found` - The resource was not found. The response body contains additional error information.

* `409 Conflict` - The request was not executed as the resource's state would become inconsistent. The response body contains additional error information. The response includes the headers: Last-Modified and ETag

```
{
	"_error": {
		"_code": "409",
		"_message": "You are trying to set an invalid country code: ABC"
	}
}
```

* `412 Pre-condition Failed` - The request was not executed as the ETag supplied did not match the resources's current ETag. The response body contains additional error information. The response includes the headers: Last-Modified and ETag

```
{
	"_error": {
		"_code": "412",
		"_message": "The resource you are trying to modify has been changed since you last read it: 3voj529d"
	}
}
```



<br/>
`DELETE /{companyId}/entities/{entityId}` - Requests the removal of the entity identified by the URL. The request must include the header: If-Match.

* `204 No Content` - The resource was removed.

* `404 Not Found` - The resource was not found. The response body contains additional error information.

* `409 Conflict` - The request was not executed as the resource's state would become inconsistent. The response body contains additional error information. The response includes the headers: Last-Modified and ETag.

* `412 Pre-condition Failed` - The request was not executed as the ETag supplied did not match the resources's current ETag. The response body contains additional error information. The response includes the headers: Last-Modified and ETag.





-----

### Items


An Item, identified by the URL `/{companyId}/items/{itemId}`, represents a specific product or service that can be sold or purchased by a [Company](#api/companies).  

Although its usage is not strictly needed, it can become convenient as a resource where your application can store additional data about products - things like EAN / UPC / ISBN product codes.

VEGAPI supports Item URLs with a fragment (the part of an URL after an `#`) chosen by your application; this fragment will be interpreted as the key of a key/value pair (a JSON object) in the Item resource, where your app can store data.

Through this facility, your app can provide support for more fine grained inventory control, enabling things like tracking expiry dates for individual lots of perishable goods, tracking inventory levels of individual sizes / colours for clothing goods or tracking individual serial numbers for electronic goods.

An Item has the following pre-defined attributes:

Name | Format | Description
---- | ------ | -----------
`_id` | string | Resource id set by VEGAPI (read only)
`_data` | object | Item attributes set by your application
`_data._name` | string | A secondary key set by your application and used in queries and lists
`_data._description` | string | Item full name
`_data._itemType` | string | Type of item for accounting purposes - see [Item Types](#settings/item-types)
`_data._tags` | array of strings | Free strings to tag this Item, defined and set by your application and mainly used to filter item lists
`_data._units`| array | List of units in which the Item may be bought / sold
`_data._units[i]` | object | Key / value pair representing the unit name and the factor to convert it into the  default unit for inventory purposes
`_data._inventoryUnit` | string | Default unit to be used in this Item's inventory; must be one of the units defined above (and its conversion factor must be 1)
`_data.inventoryCostingMethod` | string | Method used to calculate cost of goods for this item; may be "FIFO" (first in first out) or "WAVG" (Weighted Average)
`_data._vatRate` | string | VAT rate for this Item - see [VAT Rates](#settings/vat-rates)
`_lastModifiedDate` | string | Server date of the last change to resource (read only)
`_status` | string | Resource status - see [Resource states](#overview/resource-states) (read only)
`_links` | record | Links to resources related to this one (read only)
`_links._self` | string | Link to this resource (read only)
`_links._company` | string | Link to company owning this resource
`_links._documents` | string | Link to documents related to this Item



<br/>
`GET /{companyId}/items[?name={anItemName}&tags=("www", "zzz"...)&status={aStatus}]` - Requests a list of items, optionally filtered by name or tags or status.

* `200 OK` - the response body contains a list of items that satisfy the query constraints

```
{
	"_data": [
		{
			"_name": "Man Polo Shirt",
			"_id": "/5v4080jn/items/9erjg6r1"
		},
		{
			"_name": "Women Polo Shirt",
			"_id": "/5v4080jn/items/ero6j1rk"
		},
		{
			"_name": "Shipping costs",
			"_id": "/5v4080jn/items/dn0j11vy"
		}
	],
	"_links": {
		"_self": "/5v4080jn/items"
	}
}
```
	
* `206 Partial Content` - The response body contains part of the response items as indicated in the Content-Range header.
	
* `401 Unauthorized` - The response body contains an error object

```
{
	"_error": {
		"_code": "401",
		"_message": "Your applicaton needs to be authenticated to access this API"
	}
}
```



<br/>
`POST /{companyId}/items` - Requests the creation of a new item. The request body must contain a representation of the new resource or an empty JSON object - see [Resource states](#api/resource-states).

```
{
	"_data": {
		"_name": "Man Polo Shirt",
		"_description": "Man Polo Shirt - spring 2014",
		"_itemType": "Goods",
		"_tags": ["Apparel", "Summer"],
		"_units": [
			{"Pieces": 1},
			{"Cartons": 5},
			{"Boxes": 50}
			],
		"_inventoryUnit": "Pieces",
		"_vatRate": "Standard"
	}
}
```

* `201 Created` - The response includes the headers Location (with the absolute URL for the new resource), Last-Modified and ETag. The response body contains a representation of the new resource.

* `401 Unauthorized` - The response body contains an error object.




<br/>
`GET /{companyID}/items/{itemId}` - Requests the full representation of the Item identified by the URL. The request may include the header If-Modified-Since.

* `200 OK` - The response body contains the resource. The response includes the headers: Last-Modified and ETag.

```
{
    "_id": "/5v4080jn/items/9erjg6r1",
	"_data": {
		"_name": "Man Polo Shirt",
		"_description": "Man Polo Shirt - spring 2014",
		"_itemType": "Goods",
		"_tags": ["Apparel", "Summer"],
		"_units": [
			{"Pieces": 1},
			{"Cartons": 5},
			{"Boxes": 50}
			],
		"_inventoryUnit": "Pieces",
		"_vatRate": "Standard"
	},
    "_lastModifiedDate": "2014-07-21T10:37:45Z",
    "_status": "active",
	"_links": {
		"_self": "/5v4080jn/items/9erjg6r1",
        "_company": "5v4080jn",
		"_documents": "/5v4080jn/documents?item=9erjg6r1"
	}
}
```

* `304 Not Modified` - The resource has not been modified since the date indicated in the request header.

* `404 Not Found` - The resource was not found. The response body contains additional error information

```
{
	"_error": {
		"_code": "404",
		"_message": "The requested resource was not found: 3voj529d"
	}
}
```


* `410 Gone` - The resource has been deleted. The response body contains additional error information

```
{
	"_error": {
		"_code": "410",
		"_message": "This resource has been deleted and is no longer available: 3voj529d"
	}
}
```



<br/>
`PUT /{companyId}/items/{itemId}` - Requests the replacement of the Item identified by the URL. The request body must contain a representation of the new resource, where the read-only properties may be omitted (they will be ignored). The request must include the header If-Match.

```
{
	"_data": {
		"_name": "Man Polo Shirt",
		"_description": "Man Polo Shirt - spring 2014",
		"_itemType": "Goods",
		"_tags": ["Apparel", "Summer"],
		"_units": [
			{"Pieces": 1},
			{"Cartons": 5},
			{"Boxes": 50}
			],
		"_inventoryUnit": "Pieces",
		"_vatRate": "Standard"
	}
}
```

* `200 OK` - The resource was updated. The response body contains the updated resource. The response includes the headers: Last-Modified and ETag.

* `404 Not Found` - The resource was not found. The response body contains additional error information.

* `409 Conflict` - The request was not executed as the resource's state would become inconsistent. The response body contains additional error information. The response includes the headers: Last-Modified and ETag

```
{
	"_error": {
		"_code": "409",
		"_message": "Inventory unit cannot be modified after item creation"
	}
}
```


* `412 Pre-condition Failed` - The request was not executed as the ETag supplied did not match the resources's current ETag. The response body contains additional error information. The response includes the headers: Last-Modified and ETag

```
{
	"_error": {
		"_code": "412",
		"_message": "The resource you are trying to modify has been changed since you last read it: 3voj529d"
	}
}
```



<br/>
`DELETE /{companyId}/items/{itemId}` - Requests the removal of the Item identified by the URL. The request must include the header: If-Match.

* `204 No Content` - The resource was removed.

* `404 Not Found` - The resource was not found. The response body contains additional error information.

* `409 Conflict` - The request was not executed as the resource's state would become inconsistent. The response body contains additional error information. The response includes the headers: Last-Modified and ETag.

* `412 Pre-condition Failed` - The request was not executed as the ETag supplied did not match the resources's current ETag. The response body contains additional error information. The response includes the headers: Last-Modified and ETag.



-----

### Accounting-batches


An Accounting Batch, identified by the URL `/{companyId}/accounting-batches/{batchId}`, is a set of related accounting entries, either generated by VEGAPI (see [Accounting and tax](#overview/accounting-and-tax))or created by your application.  

An Accounting-batch has the following pre-defined attributes:

Name | Format | Description
---- | ------ | -----------
`_id` | string | Resource id set by VEGAPI (read only)
`_data` | object | Accounting batch attributes set by your application
`_data._name` | string | A secondary key set by your app and used for queries and in lists. In batches generated by VEGAPI it contains the name of the source document
`_data._date` | string | Date this batch was generated
`_data._description` | string | Description of this batch
`_data._journal` | string | Identification of journal where this batch is included 
`_data._source` | object | Source of this batch (optional)
`_data._source._type` | string | Type of resource. May be "Document", "Payment", "Cash-transaction" or "Auto" 
`_data._source._name` | string | Name of resource
`_data._source._id` | string | Link to resource
`_data._entries` | array | List of entries in this batch
`_data._entries[i]._entryType` | string | Entry type - must be "Debit" or "Credit"
`_data._entries[i]._ledgerAccount` | string | Ledger account - see [Settings](#settings/ledger-accounts)
`_data._entries[i]._amount` | object | Entry amount
`_lastModifiedDate` | string | Server date of the last change to resource (read only)
`_status` | string | Resource status - see [Resource states](#overview/resource-states) (read only)
`_links` | record | Links to resources related to this one (read only)
`_links._self` | string | Link to this resource
`_links._company` | string | Link to company owning this resource



<br/>
`GET /{companyId}/accounting-batches[?name={aBatchName}&dates=(yyyymmdd-yyyymmdd)&journal={aJournalName}&status={aStatus}]` - Requests a list of all accounting batches created by {companyId}, optionally filtered by name or dates and/or journal name or status. The request may include a Range header.

* `200 OK` - The response body contains a list of batches

```
{
	"_data": [
		{
			"_name": "FC 2014/247",
			"_id": "/5v4080jn/documents/erjp90r1"
		},
		{
			"_name": "FF 1233-579/14",
			"_id": "//5v4080jn/documents/ero6j1rk"
		},
		{
			"_name": "CTB 2014/012",
			"_id": "/5v4080jn/accounting-batches/e0g8gvow"
		}
	],
	"_links": {
		"_self": "/5v4080jn/accounting-batches?dates=(20140201-20140231"
	}
}
```

* `206 Partial Content` - The response body contains part of the response items as indicated in the Content-Range header.
	
* `401 Unauthorized` - The response body contains an error object

```
{
	"_error": {
		"_code": "401",
		"_message": "Your applicaton needs to be authenticated to access this API"
	}
}
```



<br/>
`POST /{companyId}/accounting-batches` - Requests the creation of a new batch of accounting entries. The request body must contain a representation of the new resource or an empty JSON object - see [Resource states](#api/resource-states).

```
{
	"_data": {
		"_name": "CTB 2014/012",
		"_date": "20140205",
		"_description": "Adjustements to IRC for July 2014",
		"_journal": "CTB",
		"_source": null,
		"_entries": [
			{
				"_entryType": "Debit",
				"_amount": {"EUR": 12347.90},
				"_ledgerAccount": "2111"
			},
			{
				"_entryType": "Credit",
				"_amount": {"EUR": 10500.00},
				"_ledgerAccount": "2112"
			},
			{
				"_entryType": "Credit",
				"_amount": {"EUR": 1847.90},
				"_ledgerAccount": "2113"
			}
		]
	}
}
```

* `201 Created` - The response includes the headers Location (with the absolute URL for the new resource), Last-Modified and ETag. The response body contains the representation of the new resource

* `401 Unauthorized` - The response body contains an error object.



<br/>
`GET /{companyID}⁄accounting-batches/{batchId}` - Requests a specific accounting-batch identified by the URL. The request may include the header If-Modified-Since.

* `200 OK` - The response body contains the resource. The response includes the headers: Last-Modified and ETag

```
{
    "_id": "/5v4080jn/accounting-batches/e0g8gvow",
	"_data": {
		"_name": "CTB 2014/012",
		"_date": "20140205",
		"_description": "Adjustements to IRC for July 2014",
		"_journal": "CTB",
		"_source": null,
		"_entries": [
			{
				"_entryType": "Debit",
				"_amount": {"EUR": 12347.90},
				"_ledgerAccount": "2111"
			},
			{
				"_entryType": "Credit",
				"_amount": {"EUR": 10500.00},
				"_ledgerAccount": "2112"
			},
			{
				"_entryType": "Credit",
				"_amount": {"EUR": 1847.90},
				"_ledgerAccount": "2113"
			}
		]
	},
    "_status": "active",
    "_lastModifiedDate": "2014-07-21T10:37:45Z",
	"_links": {
		"_self": "/5v4080jn/accounting-batches/e0g8gvow",
        "_company": "/5v4080jn"
	}
}
```

* `304 Not Modified` - The resource has not been modified since the date indicated in the request header

* `404 Not Found` - The resource was not found. The response body contains additional error information

```
{
	"_error": {
		"_code": "404",
		"_message": "The requested resource was not found: 3voj529d"
	}
}
```

* `410 Gone` - The resource has been deleted. The response body contains additional error information

```
{
	"_error": {
		"_code": "410",
		"_message": "This resource has been deleted and is no longer available: 3voj529d"
	}
}
```



<br/>
`PUT /{companyId}/accounting-batches/{batchId}` - Requests the replacement of the accounting-batch identified by the URL. The request body must contain the new representation of the resource, where the read-only properties may be omitted (they will be ignored). The request must include the header: If-Match.

```
{
	"_data": {
		"_name": "CTB 2014/012",
		"_date": "20140205",
		"_description": "Adjustements to IRC for July 2014",
		"_journal": "CTB",
		"_source": null,
		"_entries": [
			{
				"_entryType": "Debit",
				"_amount": {"EUR": 12347.90},
				"_ledgerAccount": "2111"
			},
			{
				"_entryType": "Credit",
				"_amount": {"EUR": 10500.00},
				"_ledgerAccount": "2112"
			},
			{
				"_entryType": "Credit",
				"_amount": {"EUR": 1847.90},
				"_ledgerAccount": "2113"
			}
		]
	}
}
```

* `200 OK` - The resource was updated. The response body contains the updated resource. The response includes the headers: Last-Modified and ETag.

* `404 Not Found` - The resource was not found. The response body contains additional error information

* `409 Conflict` - The request was not executed as the resource's state would become inconsistent. The response body contains additional error information. The response includes the headers: Last-Modified and ETag

```
{
	"_error": {
		"_code": "409",
		"_message": "Total debits do not equal total credits in batch: 3voj529d"
	}
}
```

* `412 Pre-condition Failed` - The request was not executed as the ETag supplied did not match the resources's current ETag. The response body contains additional error information. The response includes the headers: Last-Modified and ETag

```
{
	"_eror": {
		"_code": "412",
		"_message": "The resource you are trying to modify has been changed since you last read it"
	}
}
```



<br/>
`DELETE /{companyId}/accounting-batches/{batchId}` - Requests the removal of the accounting-batch identified by the URL. The request must include the header: If-Match.

* `204 No Content` - The resource was removed.

* `404 Not Found` - The resource was not found. The response body contains additional error information.

* `409 Conflict` - The request was not executed as the resource's state would become inconsistent. The response body contains additional error information. The response includes the headers: Last-Modified and ETag.

* `412 Pre-condition Failed` - The request was not executed as the ETag supplied did not match the resources's current ETag. The response body contains additional error information. The response includes the headers: Last-Modified and ETag.



-----

### Accounting-errors

Accounting Errors, identified by the URL `/{companyId}/accounting-errors`, is a resource listing the errors found by VEGAPI while processing documents, payments, cash-transactions or accounting-batches.  

Accounting-errors have the following pre-defined attributes:

Name | Format | Description
---- | ------ | -----------
`_data` | array | List of accounting errors (read only)
`_data[i]._name` | string | Name of the batch where this error was found - see [Accounting-batches](#api/accounting-batches)
`_data[i]._id` | string | Link to the specific batch
`_data[i]._error` | object | Object describing error
`_data[i]._error._code` | string | Error code
`_data[i]._error._message` | string | Error message
`_data[i]._error._entry` | number | Sequence number of the batch entry causing the error (if available and starting at zero)
`_links` | object | Links to resources related to this list (read only)
`_links._self` | string | Link to this resource
`_links._company` | string | Link to company owning this resource



<br/>
`GET /{companyId}/accounting-errors` - Requests the list of all accounting errors found in {companyId} documents / payments / cash-transactions / accounting-batches at the time of the last processing (see POST below). The request may include a Range header.

* `200 OK` - The response body contains a list of accounting batches with errors

```
{
	"_data": [
		{
			"_name": "CTB 2014/012",
			"_id": "/5v4080jn/accounting-batches/e0g8gvow",
			"_error": {"_code": "1234", "_message": "Some error", "_entry": 1}
			},
		{
			"_name": "FC 2014/247",
			"_id": "/5v4080jn/documents/erjp90r1",
			"_error": {"_code": "1234", "_message": "Some error"}
			}
	],
	"_links": {
		"_self": "/5v4080jn/accounting-errors"
	}
}
```

* `206 Partial Content` - The response body contains part of the response items as indicated in the Content-Range header.
	
* `401 Unauthorized` - The response body contains an error object

```
{
	"_error": {
		"_code": "401",
		"_message": "Your applicaton needs to be authenticated to access this API"
	}
}
```



<br/>
`POST /{companyId}/accounting-errors` - Requests reprocessing of all documents with accounting errors for {companyId}. The contents of the request body are ignored.


* `201 Created` - A new accounting-errors resource was created. The response body contains the new resource. The response includes the headers: Location (with the absolute URL for the new resource), Last-Modified and ETag.




-----

### Fiscal-year-ends


A Fiscal-year-end, identified by the URL `/{companyId}/fiscal-year-ends/{fiscalYearEndId}`, represents the closing of a fiscal year for a [Company](#api/companies), indicated by its start and end dates. 

A fiscal year will include all of {companyId}'s financial movements ([documents](#api/documents), [payments](#api/payments), [cash-transactions](#api/cash-transactions) and [accounting-batches](#api/accounting-batches)) with a date attribute falling within that fiscal year's start and end dates.

Closing a fiscal year triggers the creation of 2 batches of accounting entries: one to zero account balances in that same fiscal year and another to transfer those balances for the next fiscal year. It also prevents further changes to any document belonging to that fiscal year.

A fiscal year can be reopened by deleting this resource, triggering the removal of those 2 batches.  

A Fiscal-year-end has the following pre-defined attributes:

Name | Format | Description
---- | ------ | -----------
`_id` | string | Resource id set by VEGAPI (read only)
`_data` | object | Fiscal-year-end attributes set by your application
`_data._name` | string | A secondary key set by your applicaton and used in queries and lists
`_data._description` | string | Fiscal-year-end description
`_data._fiscalYearStartDate` | string | First calendar day of fiscal year (once set cannot be changed)
`_data._fiscalYeatEndDate` | string | Last calendar day of fiscal year (once set cannot be changed)
`_data._closingBatch` | string | Link to the accounting batch that closed this fiscal-year (read only)
`_data._transferBatch` | string | Link to the accounting batch that transfers initial balances for next fiscal-year (read only)
`_lastModifiedDate` | string | Server date of the last change to resource (read only)
`_status` | string | Resource status (read only) 
`_links` | record | Links to resources related to this one (read only)
`_links._self` | string | Link to this resource
`_liks._company` | string | Link to company owning this resource



<br/>
`GET /{companyId}/fiscal-year-ends[?name={aFiscalYearEndName}&status={aStatus}]` - Requests the list of fiscal-year-ends for {companyId}, optionally filtered by name. The request may include a Range header.

* `200 OK` - The response body contains the a list of resources satisfying the query constraints

```
{
	"_data": [
		{
			"_name": "FYE 2013",
			"_id": "/5v4080jn/fiscal-year-ends/k6w7wwo2"
		}
	],
	"_links": {
		"_self": "/5v4080jn/fiscal-year-ends"
	}
}
```

* `206 Partial Content` - The response body contains part of the response items as indicated in the Content-Range header.
	
* `401 Unauthorized` - The response body contains an error object

```
{
    "_error": {
        "_code": "401",
        "_message": "Your applicaton needs to be authenticated to access this API"
    }
}
```



<br/>
`POST /{companyId}/fiscal-year-ends` - Requests the creation of a new fiscal-year-end. The request body must contain a representation of the new resource or an empty JSON object - see [Resource states](#api/resource-states).

```
{
	"_data": {
		"_name": "FYE 2013",
		"_description": "Closing of fiscal year 2013",
		"_fiscalYearStartDate": "20130101",
		"_fiscalYearEndDate": "20131231"
	}
}
```

* `201 Created` - The response includes the headers Location (with the absolute URL for the new resource), Last-Modified and ETag. The response body contains a representation of the new resource.

* `401 Unauthorized` - The response body contains an error object.



<br/>
`GET /{companyID}/fiscal-year-ends/{fiscalYearEndId}` - Requests the full representation of the fiscal-year-end identified by the URL. The request may include the header If-Modified-Since.

* `200 OK` - The response body contains the resource. The response includes the headers: Last-Modified and ETag.

```
{
    "_id": "/5v4080jn/fiscal-year-ends/k6w7wwo2",
	"_data": {
		"_name": "FYE 2013",
		"_description": "Closing of fiscal year 2013",
		"_fiscalYearStartDate": "20130101",
		"_fiscalYearEndDate": "20131231",
		"_closingBatch": "5v4080jn/accounting-batches/f1h9hupx",
		"_transferBatch": "5v4080jn/accounting-batches/zam77s1tb" 
	},
    "_status": "active",
    "_lastModifiedDate": "2014-02-05T10:37:45Z",
	"_links": {
		"_self": "/5v4080jn/fiscal-year-ends/k6w7wwo2",
        "_company": "/5v4080jn"
	}
}
```

* `304 Not Modified` - The resource has not been modified since the date indicated in the request header.

* `404 Not Found` - The resource was not found. The response body contains additional error information

* `410 Gone` - The resource has been deleted. The response body contains additional error information.



<br/>
`PUT /{companyId}/fiscal-year-ends/{fiscalYearEndId}` - Requests the replacement of the fiscal-year-end identified by the URL. The request body must contain a representation of the new resource, where the read-only properties may be omitted (they will be ignored). The request must include the header: If-Match.

```
{
	"_data": {
		"_name": "FYE 2013",
		"_description": "Closing of fiscal year 2013"
	}
}
```

* `200 OK` - The resource was updated. The response body contains the updated resource. The response includes the headers: Last-Modified and ETag.

* `202 - Accepted` - The request was accepted, but its processing is not complete. The client should query the

* `404 Not Found` - The resource was not found. The response body contains additional error information.

* `409 Conflict` - The request was not executed as the resource's state would become inconsistent. The response body contains additional error information. The response includes the headers: Last-Modified and ETag.

```
{
    "_error": {
        "_code": "409",
        "_message": "You cannot modify an already closed fiscal year end: /5v4080jn/fiscal-year-ends/k6w7wwo2"
    }
}
```


* `412 Pre-condition Failed` - The request was not executed as the ETag supplied did not match the resources's current ETag. The response body contains additional error information. The response includes the headers: Last-Modified and ETag.



<br/>
`DELETE /{companyId}/fiscal-year-ends/{fiscalYearEndId}` - Requests the removal of the fiscal-year-end identified by the URL. The request must include the header: If-Match.

* `200 OK` - The resource was removed.

* `404 Not Found` - The resource was not found. The response body contains additional error information.

* `409 Conflict` - The request was not executed as the resource's state would become inconsistent. The response body contains additional error information. The response includes the headers: Last-Modified and ETag.

* `412 Pre-condition Failed` - The request was not executed as the ETag supplied did not match the resources's current ETag. The response body contains additional error information. The response includes the headers: Last-Modified and ETag.




-----

### Account statements

An Account statement, identified by the URL `/{companyId}/account-statements?ledgerAccount={aLedgerAccount}&description={aReportTitle}&dates=(yyyymmdd-yyyymmdd)`, is a temporary resource representing statement details for the ledger account and for the period specified in the query parameters.  

An Account Statement resource has the following pre-defined attributes:

Name | Format | Description
---- | ------ | -----------
`_data` | object | Account Statement report data (read only)
`_data._title` | string | Report title
`_data._startDate` | string | Start date for report period
`_data._endDate` | string | End date for report period
`_data._reportDate` | string | Date report was generated
`_data._lines` | array | Report lines
`_data._lines[i]._sequence` | number | Line sequence number
`_data._lines[i]._ledgerAccount` | string | Ledger account
`_data._lines[i]._openingBalanceDebit` | number | TBD
`_data._lines[i]._openingBalanceCredit` | number | TBD
`_data._lines[i]._monthlyTurnoverDebit` | number | TBD
`_data._lines[i]._monthlyTurnoverCredit` | number | TBD
`_data._lines[i]._closingBalanceDebit` | number | TBD
`_data._lines[i]._closingBalanceCredit` | number | TBD
`_links` | object | Empty object




<br/>
`GET /{companyId}/account-statements?ledgerAccount={aLedgerAccount}&description={aReportTitle}&dates=(yyyymmdd-yyyymmdd)` - Requests a {companyId} Account Statement report for the account and period specified.

* `200 OK` - The response body contains an Account Statement report

```
{
    "_data": {
        "_startDate": "20140101",
        "_endDate": "20140101",
        "_description": "My Company - Account Statement for 1111",
        "_reportDate": "20140219",
        "_lines": [
            {
                "_sequence": 1,
                "_ledgerAcccount": "1111",
                "_openingBalanceDebit": 10000.00,
                "_openingBalanceCredit": 10000.00,
                "_monthlyTurnoverDebit": 5000.00,
                "_monthlyTurnoverCredit": 2000.00,
                "_closingBalanceDebit": 15000.00,
                "_closingBalanceCredit": 12000.00
            }
        ]
    },
    "_links": {}
}
```

* `401 Unauthorized` - The response body contains an error object

```
{
    "_error": {
        "_code": "401",
        "_message": "Your applicaton needs to be authenticated to access this API",
        "_id": "https://docs.vegapi.org/api-guide.html#overview/security"
    }
}
```





-----

### Balance-sheets

A Balance-sheet, identified by the URL `/{companyId}/balance-sheets?description={aReportTitle}&dates=(yyyymmdd-yyyymmdd)`, is a temporary resource representing assets and liabilities details for a [Company](#api/companies) and for the period specified in the query parameters.  

A Balance-sheet resource has the following pre-defined attributes:

Name | Format | Description
---- | ------ | -----------
`_data` | object | Balance-sheet report data (read only)
`_data._title` | string | Report title
`_data._startDate` | string | Start date for report period
`_data._endDate` | string | End date for report period
`_data._reportDate` | string | Date report was generated
`_data._lines` | array | Report lines
`_data._lines[i]._sequence` | number | Line sequence number
`_data._lines[i]._ledgerAccount` | string | Ledger account
`_data._lines[i]._openingBalanceDebit` | number | TBD
`_data._lines[i]._openingBalanceCredit` | number | TBD
`_data._lines[i]._monthlyTurnoverDebit` | number | TBD
`_data._lines[i]._monthlyTurnoverCredit` | number | TBD
`_data._lines[i]._closingBalanceDebit` | number | TBD
`_data._lines[i]._closingBalanceCredit` | number | TBD
`_links` | object | Empty object




<br/>
`GET /{companyId}/balance-sheets?description={aReportTitle}&dates=(yyyymmdd-yyyymmdd)` - Requests a {companyId} Balance-sheet report for the period specified.

* `200 OK` - The response body contains a Balance-sheet report

```
{
	"_data": {
		"_startDate": "20140101",
		"_endDate": "20140101",
		"_description": "My Company - Opening Balance Sheet for FY2014",
		"_reportDate": "20140219",
		"_lines": [
			{
				"_sequence": 1,
				"_ledgerAcccount": "1111",
				"_openingBalanceDebit": 10000.00,
				"_openingBalanceCredit": 10000.00,
				"_monthlyTurnoverDebit": 5000.00,
				"_monthlyTurnoverCredit": 2000.00,
				"_closingBalanceDebit": 15000.00,
				"_closingBalanceCredit": 12000.00
			}
		]
	},
	"_links": {}
}
```

* `401 Unauthorized` - The response body contains an error object

```
{
	"_error": {
		"_code": "401",
		"_message": "Your applicaton needs to be authenticated to access this API",
		"_id": "https://docs.vegapi.org/api-guide.html#overview/security"
	}
}
```





-----

### Income-statements

An Income-statement, identified by the URL `/{companyId}/income-statements?description={aReportTitle}&dates=(yyyymmdd-yyyymmdd)`, is a temporary resource representing income and expenses details for a [Company](#api/companies) and for the period specified in the query parameters.  

An Income-statement resource has the following pre-defined properties:

Name | Format | Description
---- | ------ | -----------
`_data` | object | Income-statement report data (read only)
`_data._startDate` | string | Start date for report period
`_data._endDate` | string | End date for report period
`_data._title` | string | Report title
`_data._reportDate` | string | Date report was generated
`_data._lines` | array | Report lines
`_data._lines[i]._sequence` | number | Line sequence number
`_data._lines[i]._ledgerAccount` | string | Ledger account
`_data._lines[i]._openingBalanceDebit` | number | TBD
`_data._lines[i]._openingBalanceCredit` | number | TBD
`_data._lines[i]._monthlyTurnoverDebit` | number | TBD
`_data._lines[i]._monthlyTurnoverCredit` | number | TBD
`_data._lines[i]._closingBalanceDebit` | number | TBD
`_data._lines[i]._closingBalanceCredit` | number | TBD
`_links` | object | Links to resources related to this list
`_links._self` | string | Empty object



<br/>
`GET /{companyId}/income-statements?description={aReportTitle}&dates=(yyyymmdd-yyyymmdd)` - Requests a specific Income-statement report for {companyId}.

* `200 OK` - The response body contains an Income Statement report

```
{
	"_data": {
		"_startDate": "20140101",
		"_endDate": "20140131",
		"_description": "My Company - Income Statement for January 2014",
		"_reportDate": "20140219",
		"_lines": [
			{
				"_sequence": 1,
				"_ledgerAcccount": "6666",
				"_openingBalanceDebit": 10000.00,
				"_openingBalanceCredit": 10000.00,
				"_monthlyTurnoverDebit": 5000.00,
				"_monthlyTurnoverCredit": 2000.00,
				"_closingBalanceDebit": 15000.00,
				"_closingBalanceCredit": 12000.00
			}
		]
	},
	"_links": {}
}
```

* `401 Unauthorized` - The response body contains an error object

```
{
	"_error": {
		"_code": "401",
		"_message": "Your applicaton needs to be authenticated to access this API"
	}
}
```




-----
### Cashflow-statement

A Cashflow-statement, identified by the URL `/{companyId}/cashflow-statements?title="xxxxx"&startDate=yyyymmdd&endDate=yyyymmdd`, is a temporary resource representing cashflow details for a [Company](#api/companies) and for the period specified in the query parameters.  

A Cashflow-statement resource has the following pre-defined attributes:

Name | Format | Description
---- | ------ | -----------
`_data` | object | Cashflow-statement report data (read only)
`_data._title` | string | Report title
`_data._startDate` | string | Start date for report period
`_data._endDate` | string | End date for report period
`_data._reportDate` | string | Date report was generated
`_data._lines` | array | Report lines
`_data._lines[i]._sequence` | number | Line sequence number
`_data._lines[i]._ledgerAccount` | string | Ledger account
`_data._lines[i]._openingBalanceDebit` | number | TBD
`_data._lines[i]._openingBalanceCredit` | number | TBD
`_data._lines[i]._monthlyTurnoverDebit` | number | TBD
`_data._lines[i]._monthlyTurnoverCredit` | number | TBD
`_data._lines[i]._closingBalanceDebit` | number | TBD
`_data._lines[i]._closingBalanceCredit` | number | TBD
`_links` | object | Links to resources related to this list
`_links._self` | string | Empty object




<br/>`GET /{companyId}/cashflow-statements?description={aReportTitle}&dates=(yyyymmdd-yyyymmdd)` - Requests a {companyId} cashflow-statement for the specific period.

* `200 OK` - The response body contains a cashflow-statement report

```
{
	"_data": {
		"_startDate": "20140101",
		"_endDate": "20140131",
		"_title": "My Company - Cashflow Statement for January 2014",
		"_reportDate": "20140219",
		"_lines": [
			{
				"_sequence": 1,
				"_ledgerAcccount": "6666",
				"_openingBalanceDebit": 10000.00,
				"_openingBalanceCredit": 10000.00,
				"_monthlyTurnoverDebit": 5000.00,
				"_monthlyTurnoverCredit": 2000.00,
				"_closingBalanceDebit": 15000.00,
				"_closingBalanceCredit": 12000.00
			}
		]
	},
	"_links": {}
}
```

* `401 Unauthorized` - The response body contains an error object ([see Overview](overview.html#json_error))

```
{
	"_error": {
		"_code": "401",
		"_message": "Your applicaton needs to be authenticated to access this API"
	}
}
```



-----
### Vat-returns

A Vat-return, identified by the URL `/{companyId}/vat-returns/{vatReturnId}`, represents a submission to the Tax Authorities, by a [Company](#api/companies), of all its VAT transactions during a calendar period.  

A Vat-return resource has the following pre-defined attributes:

Name | Format | Description
---- | ------ | -----------
`_id` | string | Resource id set by VEGAPI
`_data` | object | VAT Return attributes (read only)
`_data._name` | string | A secondary key generated by your application and used in queries and lists
`_data._description` | string | Description of this VAT Return
`_data._returnType` | string | Type of VAT Return - can be "Month" or "Quarter"
`_data._startDate` | Start date for this report period
`_data._endDate` | End date for this report period
`_data._returnDate` | string | Date this return was submitted
`_data._returnStatus` | string | Status of VAT Report submission to Tax Authorities - can be "submitted" or "pending"
`_data._returnTotals` | object | Total VAT amounts in this VAT Return
`_data._returnTotals._deductible` | object | Total amount of deductible VAT 
`_data._returnTotals._settled` | object | Total amount of settled VAT
`_data._returnTotals._credits` | object | Total amount of credit adjustements
`_data._returnTotals._debits` | object | Total amount of debit adjustements
`_data._returnTotals._toPay` | object | Total amount of VAT to pay
`_data._returnTotals._toNext` | object | Total amount of VAT moved forward for next returns
`_data._returnTotals._fromPrevious` | object | Total amount of VAT from previous returns
`_data._returnTotals._usedFromPrevious` | object | Total amount of VAT used from previous returns
`_data._returnDetails` | array | List of lines in this report
`_data._returnDetails[i]._source` | string | Link to document orginating this line
`_data._returnDetails[i]._date` | string | Date of source document
`_data._returnDetails[i]._vatClass` | string | VAT Class for this line
`_data._returnDetails[i]._vatRate` | string | VAT Rate for this line
`_data._returnDetails[i]._baseAmount` | object | Amount incurring VAT
`_data._returnDetails[i]._vatAmount` | object | Amount of VAT
`_lastModifiedDate` | string | Server date of the last change to resource
`_status` | string | Resource status
`_links` | record | Links to resources related to this one (read only)
`_links._self` | string | Link to this resource
`_links._company` | string | Link to company owning this resource



<br/>
`GET /{companyId}/vat-returns[?name={a VatReturnName}&dates=(yyyymmdd-yyyymmdd)&status={aStatus}]` - Requests a list of all VAT Returns made by {companyId}, optionally filtered by return date or status. The request may include a Range header.

* `200 OK` - The response body contains a list of VAT Returns

```
{
	"_data": [
		{
			"_name": "VAT Return 2014-1",
			"_id": "/5v4080jn/vat-returns/aj680st5"
		}
	],
	"_links": {
		"_self": "/5v4080jn/vat-returns"
	}
}
```

* `206 Partial Content` - The response body contains part of the response items as indicated in the Content-Range header.
	
* `401 Unauthorized` - The response body contains an error object

```
{
	"_error": {
		"_code": "401",
		"_message": "Your applicaton needs to be authenticated to access this API"
	}
}
```



<br/>
`POST /{companyId}/vat-returns` - Requests the creation of a new Vat-return. The request body must contain identification data and the dates to be included in the new Vat-return or an empty JSON object - see [Resource states](#api/resource-states).

```
{
	"_data": {
		"_name": "VAT Return 2014-1",
		"_description": "VAT Return Q1 2014",
		"_returnType": "Month",
		"_startDate": "20140101",
		"_endDate": "20140131"
		}
	}
}
```

* `201 Created` - The response includes the headers Location (with the absolute URL for the new resource), Last-Modified and ETag.

* `401 Unauthorized` - The response body contains an error object.



<br/>
`GET /{companyID}/vat-returns/{vatReturnId}` - Requests a specific Vat-return, identified by the URL. The request may include the header If-Modified-Since.

* `200 OK` - The response body contains the resource. The response includes the headers: Last-Modified and ETag.

```
{
    "_id": "/5v4080jn/vat-returns/aj680st5",
	"_data": {
		"_name": "VAT Return 2014-1",
		"_description": "VAT Return Jan 2014",
		"_startDate": "20140101",
		"_endDate": "20140131",
		"_returnDate": "20140415",
		"_returnType": "Month",
		"_returnTotals": {
			"_deductible": {"EUR": 53.00},
			"_settled": {"EUR": 47.00},
			"_credits": {"EUR": 0},
			"_debits": {"EUR": 0},
			"_toPay": {"EUR": 0},
			"_toNext": {"EUR": 6.00},
			"_fromPrevious": {"EUR": 0},
			"_usedFromPrevious": {"EUR": 0}
		},
		"_returnStatus": "submitted",
		"_returnDetails": [
			{
				"_source": "/5v4080jn/documents/erjp90r1",
				"_date": "20140102",
				"_vatClass": "24321",
				"_vatRate": "Standard",
				"_baseAmount": {"EUR": 100.00},
				"_vatAmount": {"EUR": 23.50}
			},
			{
				"_source": "/5v4080jn/documents/erjp90r1",
				"_date": "20140102",
				"_vatClass": "24321",
				"_vatRate": "Standard",
				"_baseAmount": {"EUR": 100.00},
				"_vatAmount": {"EUR": 23.50}
			},
			{
				"_source": "/5v4080jn/documents/erjp90r1",
				"_date": "20140102",
				"_vatClass": "24321",
				"_vatRate": "Reduced",
				"_baseAmount": {"EUR": 100.00},
				"_vatAmount": {"EUR": 6.00}
			},
			{
				"_source": "/5v4080jn/documents/erjp90r1",
				"_date": "20140102",
				"_vatClass": "24331",
				"_vatRate": "Standard",
				"_baseAmount": {"EUR": 100.00},
				"_vatAmount": {"EUR": 23.50}
			},
			{
				"_source": "/5v4080jn/documents/erjp90r1",
				"_date": "20140102",
				"_vatClass": "24331",
				"_vatRate": "Standard",
				"_baseAmount": {"EUR": 100.00},
				"_vatAmount": {"EUR": 23.50}
			}
		]
	},
    "_status": "active",
    "_lastModifiedDate": "2014-07-21T10:37:45Z",
	"_links": {
		"_self": "/5v4080jn/vat-returns/aj680st5",
        "_company": "/5v4080jn"
	}
}
```

* `304 Not Modified` - The resource has not been modified since the date indicated in the request header.

* `404 Not Found` - The resource was not found. The response body contains additional error information

```
{
	"_error": {
		"_code": "404",
		"_message": "The requested resource was not found: 3voj529d"
	}
}
```

* `410 Gone` - The resource has been deleted. The response body contains additional error information

```
{
	"_error": {
		"_code": "410",
		"_message": "This resource has been deleted and is no longer available: 3voj529d"
	}
}
```



<br/>
`DELETE /{companyId}/vat-returns/{vatReturnId}` - Requests removal of the Vat-return identified by the URL. The request must include the header: If-Match.

* `204 No Content` - The resource was removed.

* `404 Not Found` - The resource was not found. The response body contains additional error information.

* `409 Conflict` - The request was not executed as the resource's state would become inconsistent. The response body contains additional error information. The response includes the headers: Last-Modified and ETag.

* `412 Pre-condition Failed` - The request was not executed as the ETag supplied did not match the resources's current ETag. The response body contains additional error information. The response includes the headers: Last-Modified and ETag.




-----
### Drafts

Drafts, identified by the URL `/{companyId}/drafts`, represent the set of new versions of resources that are being created or updated by you application, but not yet ready to be committed to the API for further processing - basically a store for work in progress, allowing your application to implement simple segregation of duties or complex resource lifecycle strategies.

Resources posted to `/{companyId}/drafts` are stored by the API but not subject to validation or further processing. When your application wants to commit a new or an updated resource to the API, it must do it through the appropriate resource URL.

As Drafts can be of any resource type - Companies, Documents, Payments, etc - a draft representation depends on the underlying resource type. However Drafts still have to obey the same general design rules - see [Resource representation](#overview/resource-representation) above.

Name | Format | Description
---- | ------ | -----------
`_id` | string | Resource id set by VEGAPI (read only)
`_data` | object | Resource attributes set by your application
`_data._name` | string | A secondary key set by your applicaton and used in queries and lists
`data...` | ... | Any other relevant attributes for the particular resource
`_lastModifiedDate` | string | Server date of the last change to resource (read only)
`_status` | string | Resource status (read only) 
`_links` | record | Links to resources related to this one (read only)
`_links._self` | string | Link to this resource
`_links._company` | string | Link to company owning this resource



<br/>
`GET /{companyId}/drafts[?type={aResourceType}&name={aDraftName}&status={aStatus}]` - Requests the list of drafts for {companyId}, optionally filtered by resource type and/or name or status. The request may include a Range header.

* `200 OK` - The response body contains the a list of resources satisfying the query constraints

```
{
	"_data": [
		{
			"_name": "A draft fiscal year end",
			"_id": "/5v4080jn/drafts/uj27mtp5"
		}
	],
	"_links": {
		"_self": "/5v4080jn/drafts"
	}
}
```

* `206 Partial Content` - The response body contains part of the response items as indicated in the Content-Range header.
	
* `401 Unauthorized` - The response body contains an error object

```
{
    "_error": {
        "_code": "401",
        "_message": "Your applicaton needs to be authenticated to access this API"
    }
}
```



<br/>
`POST /{companyId}/drafts` - Requests the creation of a new draft. The request body must contain a representation of the new resource or an empty JSON object - see [Resource states](#api/resource-states).

```
{
	"_data": {
		"_name": "FYE 2013",
		"_description": "Closing of fiscal year 2013",
		"_fiscalYearStartDate": "20130101",
		"_fiscalYearEndDate": "20131231"
	}
}
```

* `201 Created` - The response includes the headers Location (with the absolute URL for the new resource), Last-Modified and ETag. The response body contains a representation of the new resource.

* `401 Unauthorized` - The response body contains an error object.



<br/>
`GET /{companyID}/drafts/{draftId}` - Requests the full representation of the draft identified by the URL. The request may include the header If-Modified-Since.

* `200 OK` - The response body contains the resource. The response includes the headers: Last-Modified and ETag.

```
{
    "_id": "/5v4080jn/drafts/uj27mtp5",
	"_data": {
		"_name": "FYE 2013",
		"_description": "Closing of fiscal year 2013",
		"_fiscalYearStartDate": "20130101",
		"_fiscalYearEndDate": "20131231" 
	},
    "_status": "active",
    "_lastModifiedDate": "2014-02-05T10:37:45Z",
	"_links": {
		"_self": "/5v4080jn/drafts/uj27mtp5",
        "_company": "/5v4080jn"
	}
}
```

* `304 Not Modified` - The resource has not been modified since the date indicated in the request header.

* `404 Not Found` - The resource was not found. The response body contains additional error information

* `410 Gone` - The resource has been deleted. The response body contains additional error information.



<br/>
`PUT /{companyId}/drafts/{draftId}` - Requests the replacement of the draft identified by the URL. The request body must contain a full representation of the new resource with all its properties. The request must include the header: If-Match.

```
{
	"_data": {
		"_name": "FYE 2013",
		"_description": "Closing of fiscal year 2013",
		"_fiscalYearStartDate": "20130101",
		"_fiscalYearEndDate": "20131231" 
	}
}
```

* `200 OK` - The resource was updated. The response body contains the updated resource. The response includes the headers: Last-Modified and ETag.

* `404 Not Found` - The resource was not found. The response body contains additional error information.

* `412 Pre-condition Failed` - The request was not executed as the ETag supplied did not match the resources's current ETag. The response body contains additional error information. The response includes the headers: Last-Modified and ETag.



<br/>
`DELETE /{companyId}/drafts/{draftId}` - Requests the removal of the draft identified by the URL. The request must include the header: If-Match.

* `200 OK` - The resource was removed.

* `404 Not Found` - The resource was not found. The response body contains additional error information.

* `412 Pre-condition Failed` - The request was not executed as the ETag supplied did not match the resources's current ETag. The response body contains additional error information. The response includes the headers: Last-Modified and ETag.




-----

### Users


A User, identified by the URL `/{companyId}/users/{userId}`, is any person allowed to access [Company](#api/companies) resources, as specified in a separate [Profile](#api/profiles).  

Although its usage is not strictly needed, it can become convenient as a resource where your application can store data about its users.  

A User has the following pre-defined attributes:


Name | Format | Description
---- | ------ | -----------
`_id` | string | Resource id set by VEGAPI (read only)
`_data._name` | string | A secondary key set by your application and used in queries and lists
`_data._description` | string | User full name
`_data._accessProfile` | object | Access profile of this user
`_data._accessProfile._name` | string | Name of access profile
`_data._accessProfile._href` | string | Link to access profile
`_lastModifiedDate` | string | Server date of the last change to resource, in [ISO 8601 format](https://en.wikipedia.org/wiki/ISO_8601) (read only)
`_status` | string | Resource status - see [Resource states](#overview/resource-states) (read only)
`_links` | record | Links to resources related to this one (read only)
`_links._self` | string | Link to this resource
`_links._company` | string | Link to company owning this resource
`_links._profiles` | string | Link to access profiles of this Company



<br/>
`GET /{companyId}/users[?name={aUserName}&status={aStatus}]` - Requests a list of all users created by {companyId}, optionally filtered by name or status. The request may include a Range header.

* `200 OK` - the response body contains a list of users satisfying the query constraints

```
{
    "_data": [
        {
            "_name": "User1",
            "_href": "/5v4080jn/users/4xrzmv58"
        },
        {
            "_name": "User2",
            "_href": "/5v4080jn/users/k05qx909"
        }
    ],
    "_links": {
        "_self": "/5v4080jn/users"
    }
}
```
    
* `206 Partial Content` - The response body contains part of the response items as indicated in the Content-Range header.
    
* `401 Unauthorized` - The response body contains an error object

```
{
    "_error": {
        "_code": "401",
        "_message": "Your applicaton needs to be authenticated to access this API"
    }
}
```



<br/>
`POST /{companyId}/users` - Requests the creation of a new user. The request body must contain a representation of the new resource or an empty JSON object - see [Resource states](#api/resource-states).

```
{
    "_data": {
        "_id": "4xrzmv58",
        "_name": "User1",
        "_descripton": "User Full Name",
        "_accessProfile": {
            "_name": "admin",
            "_href": "/5v4080jn/profiles/gs608wca"
        }
    }
}
```

* `201 Created` - The response includes the headers Location (with the absolute URL for the new resource), Last-Modified and ETag. The response body contains a representation of the new resource.

* `401 Unauthorized` - The response body contains an error object.



<br/>
`GET /{companyID}/users/{userId}` - Requests the full representation of the user identified by the URL. The request may include the header If-Modified-Since.

* `200 OK` - The response body contains the resource. The response includes the headers: Last-Modified and ETag.

```
{
    "_data": {
        "_id": "4xrzmv58",
        "_name": "User1",
        "_descripton": "User Full Name",
        "_accessProfile": {
            "_name": "admin",
            "_href": "/5v4080jn/profiles/gs608wca"
        },
    },
    "_status": "active",
    "_lastModifiedDate": "2014-07-21T10:37:45Z",
    "_links": {
        "_self": "/5v4080jn/users/4xrzmv58",
        "_company": "/5v4080jn",
        "_profiles": "/5v4080jn/profiles"
    }
}
```

* `304 Not Modified` - The resource has not been modified since the date indicated in the request header.

* `404 Not Found` - The resource was not found. The response body contains additional error information

```
{
    "_error": {
        "_code": "404",
        "_message": "The requested resource was not found: 3voj529d"
        }
}
```

* `410 Gone` - The resource has been deleted. The response body contains additional error information

```
{
    "_error": {
        "_code": "410",
        "_message": "This resource has been deleted and is no longer available: 3voj529d"
    }
}
```



<br/>
`PUT /{companyId}/users/{userId}` - Requests the replacement of the user identified by the URL. The request body must contain a representation of the new resource, where the read-only properties may be omitted (they will be ignored). The request must include the header: If-Match.

```
{
    "_data": {
        "_id": "4xrzmv58",
        "_name": "User1",
        "_descripton": "User Full Name",
        "_accessProfile": {
            "_name": "admin",
            "_href": "/5v4080jn/profiles/gs608wca"
        }
    }
}
```

* `200 OK` - The resource was updated. The response body contains the updated resource. The response includes the headers: Last-Modified and ETag.

* `404 Not Found` - The resource was not found. The response body contains additional error information.

* `409 Conflict` - The request was not executed as the resource's state would become inconsistent. The response body contains additional error information. The response includes the headers: Last-Modified and ETag

```
{
    "_error": {
        "_code": "409",
        "_message": "You are trying to set an invalid country code: ABC"
    }
}
```

* `412 Pre-condition Failed` - The request was not executed as the ETag supplied did not match the resources's current ETag. The response body contains additional error information. The response includes the headers: Last-Modified and ETag

```
{
    "_error": {
        "_code": "412",
        "_message": "The resource you are trying to modify has been changed since you last read it: 3voj529d"
    }
}
```



<br/>
`DELETE /{companyId}/users/{userId}` - Requests the removal of the user identified by the URL. The request must include the header: If-Match.

* `204 No Content` - The resource was removed.

* `404 Not Found` - The resource was not found. The response body contains additional error information.

* `409 Conflict` - The request was not executed as the resource's state would become inconsistent. The response body contains additional error information. The response includes the headers: Last-Modified and ETag.

* `412 Pre-condition Failed` - The request was not executed as the ETag supplied did not match the resources's current ETag. The response body contains additional error information. The response includes the headers: Last-Modified and ETag.





-----

### Profiles


A Profile, identified by the URL `/{companyId}/profiles/{profileId}`, is an optional resource detailing a set of access rights to [Company](#api/companies) resources, applicable to that Company [Users](#api/users).  

Although its usage is not strictly needed, it can become convenient as a resource where your application can store data about its access profiles.  

A Profile has the following pre-defined attributes:


Name | Format | Description
---- | ------ | -----------
`_id` | string | Resource id set by VEGAPI (read only)
`_data._name` | string | A secondary key set by your application and used in queries and lists
`_data._description` | string | Profile full name
`_data._resourceAccess` | object | Resources authorized for this profile
`_data._resourceAccess.{key}` | string | Name of resource
`_data._resourceAccess.{value}` | string | Type of access authorized - can be 'none', 'read-only' or 'read-write'
`_lastModifiedDate` | string | Server date of the last change to resource, in [ISO 8601 format](https://en.wikipedia.org/wiki/ISO_8601) (read only)
`_status` | string | Resource status - see [Resource states](#overview/resource-states) (read only)
`_links` | record | Links to resources related to this one (read only)
`_links._self` | string | Link to this resource
`_links._company` | string | Link to company owning this resource
`_links._profiles` | string | Link to access profiles of this Company



<br/>
`GET /{companyId}/profiles[?name={aProfileName}&status={aStatus}]` - Requests a list of all profiles created by {companyId}, optionally filtered by name or status. The request may include a Range header.

* `200 OK` - the response body contains a list of profiles satisfying the query constraints

```
{
    "_data": [
        {
            "_name": "admin",
            "_href": "/5v4080jn/profiles/gs608wca"
        },
        {
            "_name": "toc",
            "_href": "/5v4080jn/profiles/4a3a7bhf"
        }
    ],
    "_links": {
        "_self": "/5v4080jn/profiles"
    }
}
```
    
* `206 Partial Content` - The response body contains part of the response items as indicated in the Content-Range header.
    
* `401 Unauthorized` - The response body contains an error object

```
{
    "_error": {
        "_code": "401",
        "_message": "Your applicaton needs to be authenticated to access this API"
    }
}
```



<br/>
`POST /{companyId}/profiles` - Requests the creation of a new profile. The request body must contain a representation of the new resource or an empty JSON object - see [Resource states](#api/resource-states).

```
{
    "_data": {
        "_id": "gs608wca",
        "_name": "admin",
        "_descripton": "Administrator",
        "_resourceAccess": {
            "companies": "read-write",
            "entities": "read-write",
            "items": "read-write",
            "documents": "read-write",
            "payments": "read-write",
            "cash-transactions": "read-write",
            "accounting-batches": "read-write",
            "accounting-errors": "read-write",
            "fiscal-years": "read-write",
            "vat-returns": "read-write",
            "users": "read-write",
            "profiles": "read-write",
            "settings": "read-write"
        }
    }
}
```

* `201 Created` - The response includes the headers Location (with the absolute URL for the new resource), Last-Modified and ETag. The response body contains a representation of the new resource.

* `401 Unauthorized` - The response body contains an error object.



<br/>
`GET /{companyID}/profiles/{profileId}` - Requests the full representation of the profile identified by the URL. The request may include the header If-Modified-Since.

* `200 OK` - The response body contains the resource. The response includes the headers: Last-Modified and ETag.

```
{
    "_data": {
        "_id": "gs608wca",
        "_name": "admin",
        "_descripton": "Administrator",
        "_resourceAccess": {
            "companies": "read-write",
            "entities": "read-write",
            "items": "read-write",
            "documents": "read-write",
            "payments": "read-write",
            "cash-transactions": "read-write",
            "accounting-batches": "read-write",
            "accounting-errors": "read-write",
            "fiscal-years": "read-write",
            "vat-returns": "read-write",
            "users": "read-write",
            "profiles": "read-write",
            "settings": "read-write"
        },
    "_status": "active",
    "_lastModifiedDate": "2014-07-21T10:37:45Z"
    },
    "_links": {
        "_self": "/5v4080jn/profiles/gs608wca",
        "_companies": "/5v4080jn",
        "_users": "5v4080jn/users"
    }
}
```

* `304 Not Modified` - The resource has not been modified since the date indicated in the request header.

* `404 Not Found` - The resource was not found. The response body contains additional error information

```
{
    "_error": {
        "_code": "404",
        "_message": "The requested resource was not found: 3voj529d"
        }
}
```

* `410 Gone` - The resource has been deleted. The response body contains additional error information

```
{
    "_error": {
        "_code": "410",
        "_message": "This resource has been deleted and is no longer available: 3voj529d"
    }
}
```



<br/>
`PUT /{companyId}/profiles/{profileId}` - Requests the replacement of the profile identified by the URL. The request body must contain a representation of the new resource, where the read-only properties may be omitted (they will be ignored). The request must include the header: If-Match.

```
{
    "_data": {
        "_id": "gs608wca",
        "_name": "admin",
        "_descripton": "Administrator",
        "_resourceAccess": {
            "companies": "read-write",
            "entities": "read-write",
            "items": "read-write",
            "documents": "read-write",
            "payments": "read-write",
            "cash-transactions": "read-write",
            "accounting-batches": "read-write",
            "accounting-errors": "read-write",
            "fiscal-years": "read-write",
            "vat-returns": "read-write",
            "users": "read-write",
            "profiles": "read-write",
            "settings": "read-write"
        }
    }
}
```

* `200 OK` - The resource was updated. The response body contains the updated resource. The response includes the headers: Last-Modified and ETag.

* `404 Not Found` - The resource was not found. The response body contains additional error information.

* `409 Conflict` - The request was not executed as the resource's state would become inconsistent. The response body contains additional error information. The response includes the headers: Last-Modified and ETag

```
{
    "_error": {
        "_code": "409",
        "_message": "You are trying to set an invalid country code: ABC"
    }
}
```

* `412 Pre-condition Failed` - The request was not executed as the ETag supplied did not match the resources's current ETag. The response body contains additional error information. The response includes the headers: Last-Modified and ETag

```
{
    "_error": {
        "_code": "412",
        "_message": "The resource you are trying to modify has been changed since you last read it: 3voj529d"
    }
}
```



<br/>
`DELETE /{companyId}/profiles/{profileId}` - Requests the removal of the profile identified by the URL. The request must include the header: If-Match.

* `204 No Content` - The resource was removed.

* `404 Not Found` - The resource was not found. The response body contains additional error information.

* `409 Conflict` - The request was not executed as the resource's state would become inconsistent. The response body contains additional error information. The response includes the headers: Last-Modified and ETag.

* `412 Pre-condition Failed` - The request was not executed as the ETag supplied did not match the resources's current ETag. The response body contains additional error information. The response includes the headers: Last-Modified and ETag.





-----
## Settings


Settings, identified by the URL `/{companyId}/settings`, represents the collection of parameter settings used by VEGAPI to drive accounting and tax processing for a [Company](#api/companies).  

<br/>
The following settings are pre-defined in VEGAPI:
* cash-accounts
* document-types
* item-types
* ledger-accounts
* instruments
* payment-types
* vat-classes
* vat-rates

<br/>
All these individual settings follow the same structure:
* all are singleton resources - only one resource exists for each
* all support the same HTTP methods: GET and PUT - you can't delete a settings resource
* your application can create a new settings resource with a `_name` selected by you - **be careful to avoid overwriting pre-defined VEGAPI settings**




<br/>
`GET /{companyId}/settings` - Requests a list of all settings available for {companyId}.

* `200 OK` - the response body contains an attribute (`_links`) with links to specific settings

```
{
	"_data": {},
	"_links": {
		"_self": "/5v4080jn/settings",
        "_company": "/5v4080jn",
        "_cashAccounts": "/5v4080jn/settings/cash-accounts",
        "_documentTypes": "/5v4080jn/settings/document-types",
        "_itemTypes": "/5v4080jn/settings/item-types",
        "_ledgerAccounts": "/5v4080jn/settings/ledger-accounts",
        "_paymentInstruments": "/5v4080jn/settings/instruments"
        "_paymentTypes": "/5v4080jn/settings/payment-types"
        "_vatClasses": "/5v4080jn/settings/vat-classes",
        "_vatRates": "/5v4080jn/settings/vat-rates"
	}
}
```

* `401 Unauthorized` - The response body contains an error object

```
{
	"_error": {
		"_code": "401",
		"_message": "Your applicaton needs to be authenticated to access this API"
	}
}
```



-----
### Cash-accounts

Cash-accounts, identified by the URL `/{companyId}/settings/cash-accounts`, contains accounting information required by VEGAPI to process cash-accounts and cash-transactions used by {companyId}.  

Cash-accounts can be of type "Cashier" (internal) or "Bank" (external) and must have a ledger-account associated with them.  


Name | Format | Description
---- | ------ | -----------
`_data` | object | Cash-account settings object 
`_data.<name>` | object | Settings for cash-account named <*name*>
`_data.<name>._cashAccountType` | string | Type of cash-account. Can be "Cashier" or "Bank"
`_data.<name>._ledgerAccount` | string | Name of ledger account associated with this cash-account
`_lastModifiedDate` | string | Server date of the last change to resource (read only)
`_status` | string | Resource status - see [Resource states](#overview/resource-states) (read only)
`_links` | object | Links to related resources (read only)
`_links._self` | string | Link to this resource
`_links._company` | Link to company owning this resource
`_links._settings` | string | Link to {companyId} settings



<br/>
A sample cash-account resource would be:


```
{
	"_data": {
		"Cashier HO": {
			"_cashAccountType": "Cashier",
			"_ledgerAccount": "11"
		},
		"BPI DO": {
			"_cashAccountType": "Bank",
			"_ledgerAccount": "12"
		}
	},
    "_status": "active",
    "_lastModifiedDate": "2014-07-21T10:37:45Z",
	"_links": {
		"_self": "/5v4080jn/settings/cash-accounts",
        "_company": "/5v4080jn",
		"_settings": "/5v4080jn/settings"
	}
}
```



-----
### Document-types

Document-types, identified by the URL `/{companyId}/settings/document-types`, contains information used by VEGAPI for the accounting and tax processing of {companyId} documents.

**ToDo**: criar descrição  
 

Name | Format | Description
---- | ------ | -----------
`_data` | object | Settings object 
`_data.<name>` | object | Settings for document-type <*name*>
`_data.<name>._description` | string | Document name
`_lastModifiedDate` | string | Server date of the last change to resource (read only)
`_status` | string | Resource status - see [Resource states](#overview/resource-states) (read only)
`_links` | object | Links to related resources (read only)
`_links._self` | string | Link to this resource
`_links._settings` | string | Link to {companyId} settings


<br/>
A sample document-type resource would be:

```
{
}
```


-----
### Item-types

Item-types, identified by the URL `/{companyId}/settings/payment-types`, contains information used by VEGAPI in the accounting and tax processing of the line items contained in {companyId} documents.  
 
**ToDo**: criar descrição

Name | Format | Description
---- | ------ | -----------
`_data` | object | Settings object 
`_data.<name>` | object | Settings for item-type <*name*>
`_data.<name>._description` | string | 
`_lastModifiedDate` | string | Server date of the last change to resource (read only)
`_status` | string | Resource status 
`_links` | object | Links to related resources (read only)
`_links._self` | string | Link to this resource
`_links._company` | string | Link to company owning this resource
`_links._settings` | string | Link to {companyId} settings




-----
### Ledger-accounts

The Ledger-accounts resource, identified by the URL `/{companyId}/settings/ledger-accounts`, represents the set of {companyId} ledger accounts, used by VEGAPI in the accounting and tax processing of documents, payments, cash-transactions and accounting-batches.  

**ToDo**: criar descrição

Name | Format | Description
---- | ------ | -----------
`_data` | object | Settings object 
`_data.<name>` | object | Settings for ledger-account <*name*>
`_data.<name>._description` | string | Ledger-account description
`_data.<name>._acceptsTransactions` | boolean | If false, account is only for reporting purposes - no entries can be posted to it
`_lastModifiedDate` | string | Server date of the last change to resource (read only)
`_status` | string | Resource status 
`_links` | object | Links to related resources (read only)
`_links._self` | string | Link to this resource
`_links._company` | string | Link to company owning this resource
`_links._settings` | string | Link to {companyId} settings



-----
### Instruments


Instruments, identified by the URL `/{companyId}/settings/payment-instruments`, contains all valid instruments used by {companyId} in its payments and financial transactions.  

Instruments can be of type "Cash" (notes and coins), "Cheque", "Bank transfer", "Debit Card", "Credit Card" and "Paypal".   

Name | Format | Description
---- | ------ | -----------
`_data` | object | Settings object 
`_data.<name>` | object | Settings for instrument <*name*>
`_lastModifiedDate` | string | Server date of the last change to resource (read only)
`_status` | string | Resource status - see [Resource states](overview#resource-states) 
`_links` | object | Links to related resources (read only)
`_links._self` | string | Link to this resource
`_links._company` | string | Link to company owning this resource
`_links._settings` | string | Link to {companyId} settings


<br/>
A sample instruments resource would be:

```
{
	"_data": {
		"Cash": "",
		"Cheque": "",
		"BankTransfer": "",
		"Credit/DebitCard": "",
		"Paypal": ""
	},
    "_status": "active",
    "_lastModifiedDate": "2014-07-21T10:37:45Z",
	"_links": {
		"_self": "/5v4080jn/settings/payment-instruments",
        "_company": "/5v4080jn"
	}
}
```




-----
### Payment-types

Payment-types is a resource, identified by the URL `/{companyId}/settings/payment-types`, representing the valid payment-types for {companyId}. Each payment-type is represented by a key/value pair, where the key is defined by your application and the value is an object with attributes for that payment-type, namely a ???.  
 

Name | Format | Description
---- | ------ | -----------
`_data` | object | Settings object 
`_data.<name>` | object | Settings for payment-type <*name*>
`_data.<name>._description` | string | Payment-type description
`_data.<name>._????` | number | ????
`_lastModifiedDate` | string | Server date of the last change to resource (read only)
`_status` | string | Resource status (read only) 
`_links` | object | Links to related resources (read only)
`_links._self` | string | Link to this resource
`_links._company` | string | Link to company owning this resource
`_links._settings` | string | Link to {companyId} settings




-----
### VAT-classes

Vat-classes is a resource, identified by the URL `/{companyId}/settings/vat-classes`, representing the valid VAT classes for {companyId}. Each Vat-class is represented by a key/value pair, where the key is defined by your application and the value is an object with attributes, namely a description, the ledger-account to be used for this VAT Class and the matching VAT Class to be used for reversions.  
 

Name | Format | Description
---- | ------ | -----------
`_data` | object | Settings object 
`_data.<name>` | object | Settings for vat-class <*name*>
`_data.<name>._description` | string | VAT Class description
`_data.<name>._ledgerAccount` | string | Ledger Account to be used
`_data.<name>._vatClassForReversions` | string | VAT Class to be used when reverting entries 
`_lastModifiedDate` | string | Server date of the last change to resource (read only)
`_status` | string | Resource status (read only)
`_links` | object | Links to related resources (read only)
`_links._self` | string | Link to this resource
`_links._company` | string | Link to company owning this resource
`_links._settings` | string | Link to {companyId} settings




-----
### VAT-rates

VAT-rates is a resource, identified by the URL `/{companyId}/settings/vat-rates`, representing the valid VAT Rates for {companyId}. Each VAT-rate is a key/value pair, where the key is defined by your application and the value is an object with attributes for that VAT Rate, namely a description and the rate value in decimal format.  
 
Name | Format | Description
---- | ------ | -----------
`_data` | object | Settings object 
`_data.<name>` | object | Settings for vat-rate <*name*>
`_data.<name>._description` | string | VAT Rate description
`_data.<name>._value` | number | Rate value in decimal
`_lastModifiedDate` | string | Server date of the last change to resource (read only)
`_status` | string | Resource status (read only)
`_links` | object | Links to related resources (read only)
`_links._self` | string | Link to this resource
`_links._company` | string | Link to company owning this resource
`_links._settings` | string | Link to {companyId} settings





## System


-----
### Audit-logs

An Audit-log, identified by the URL `/audit-logs/{fileName}`, is a file containing a log of all HTPP interactions between your application and VEGAPI for a period of approximately 2 hours.    

An Audit-log record is a JSON object with the following attributes:

Name | Format | Description
---- | ------ | -----------
`name` | string | Log name, taken from VEGAPI instance name - see [Resource identification](#overview/resource-identification)
`hostname` | string | VEGAPI host identification
`pid` | string | VEGAPI process identification
`audit` | boolean | Generate audit records? Default is true
`level` | number | Loggin level. 
`remoteAddress` | string | IP address of remote client
`remotePort` | number | Port number of remote client
`req_id` | string | Unique request identifier assigned to each VEGAPI request
`req` | object | Client request to VEGAPI
`req.query` | object | Query part of request URL
`req.method` | string | Request method
`req.url` | string | Request URL
`req.headers` | object | Request headers
`req.httpVersion` | string | Request HTTP version
`req.trailers` | object | Request trailers
`req.version` | string | Logging software version
`req.body` | object | Request body
`res` | object | VEGAPI response to client
`res.StatusCode` | number | Response HTTP status code
`res.headers` | object | Response headers 
`res.trailers` | object | Response trailers 
`res.body` | object | Response body 
`latency` | number | Latency of this request in ms
`msg` | string | Text message
`time` | string | Log creation timestamp
`v` | number | Logging software version




<br/>`GET /audit-logs` - Displays the list of available audit-logs.

* `200 OK` - The response body contains a list of available audit logs

```
{
  "_data": [
    {
      "_id": "audit.log",
      "_lastModifiedDate": "2015-07-13T18:55:44.397Z"
    },
    {
      "_id": "audit.log.0",
      "_lastModifiedDate": "2015-07-13T17:51:15.925Z"
    },
    {
      "_id": "audit.log.1",
      "_lastModifiedDate": "2015-07-13T15:47:54.881Z"
    },
    {
      "_id": "audit.log.10",
      "_lastModifiedDate": "2015-07-12T21:39:56.957Z"
    },
    {
      "_id": "audit.log.2",
      "_lastModifiedDate": "2015-07-13T12:19:04.953Z"
    },
    {
      "_id": "audit.log.3",
      "_lastModifiedDate": "2015-07-13T11:35:43.953Z"
    },
    {
      "_id": "audit.log.4",
      "_lastModifiedDate": "2015-07-13T08:50:51.497Z"
    },
    {
      "_id": "audit.log.5",
      "_lastModifiedDate": "2015-07-13T07:40:01.217Z"
    },
    {
      "_id": "audit.log.6",
      "_lastModifiedDate": "2015-07-13T04:00:00.093Z"
    },
    {
      "_id": "audit.log.7",
      "_lastModifiedDate": "2015-07-13T02:30:26.981Z"
    },
    {
      "_id": "audit.log.8",
      "_lastModifiedDate": "2015-07-13T01:01:48.805Z"
    },
    {
      "_id": "audit.log.9",
      "_lastModifiedDate": "2015-07-12T22:00:00.097Z"
    }
  ],
  "_links": {
    "_self": "/audit-logs"
  }
}
```

* `401 Unauthorized` - The response body contains an error object ([see Overview](overview.html#json_error))



<br/>
`GET /{companyID}/audit-logs/{auditLogId}` - Requests the full representation of the audit log identified by the URL. The request may include the header If-Modified-Since.

* `200 OK` - The response body contains the resource. The response includes the headers: Last-Modified and ETag.

```
{
    "name": "test",
    "hostname": "ip-172-31-38-27",
    "pid": 20320,
    "audit": true,
    "level": 30,
    "remoteAddress": "::ffff:94.60.252.25",
    "remotePort": 53390,
    "req_id": "7a09992b-2587-4107-8543-4983c9cdd2ff",
    "req": {
        "query": {},
        "method": "GET",
        "url": "/41MKs_isO",
        "headers": {
            "host": "test.vegapi.org:8080",
            "connection": "keep-alive",
            "accept": "application/json",
            "cache-control": "no-cache",
            "user-agent": "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_10_4) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/43.0.2357.132 Safari/537.36",
            "csp": "active",
            "authorization": "Basic xxxxxxxxxxxxxxxx",
            "postman-token": "8f5a609d-98e6-b168-d4c0-1b4e4a0c20ab",
            "accept-encoding": "gzip, deflate, sdch",
            "accept-language": "en,en-US;q=0.8,pt-PT;q=0.6,pt;q=0.4"
        },
        "httpVersion": "1.1",
        "trailers": {},
        "version": "*"
    },
    "res": {
        "statusCode": 200,
        "headers": {
            "content-encoding": "gzip",
            "content-type": "application/json"
        },
        "trailer": false,
        "body": {
            "_id": "/41MKs_isO",
            "_data": {
                "_name": "My Company",
                "_description": "My Company, Lda.",
                "_taxNumber": "PTXXX",
                "_country": "PT",
                "_currency": "EUR",
                "_addresses": {
                    "_main": "Ainda outra morada qualquer"
                },
                "armazém": 123
            },
            "_status": "active",
            "_lastModifiedDate": "2015-07-12T21:34:57.571Z",
            "_links": {
                "_self": "/41MKs_isO",
                "_company": "/41MKs_isO",
                "_documents": "/41MKs_isO/documents",
                "_payments": "/41MKs_isO/payments",
                "_cashTransactions": "/41MKs_isO/cashTransactions",
                "_entities": "/41MKs_isO/entities",
                "_items": "/41MKs_isO/items",
                "_accountingBatches": "/41MKs_isO/accountingBatches",
                "_accountingErrors": "/41MKs_isO/accountingErrors",
                "_fiscalYearEnds": "/41MKs_isO/fiscalYearEnds",
                "_accountStatements": "/41MKs_isO/accountStatements",
                "_balanceSheets": "/41MKs_isO/balanceSheets",
                "_incomeStatements": "/41MKs_isO/incomeStatements",
                "_cashflowStatements": "/41MKs_isO/cashflowStatements",
                "_vatReturns": "/41MKs_isO/vatReturns",
                "_drafts": "/41MKs_isO/drafts",
                "_users": "/41MKs_isO/users",
                "_profiles": "/41MKs_isO/profiles",
                "_settings": "/41MKs_isO/settings"
            }
        }
    },
    "latency": 5,
    "_audit": true,
    "msg": "Handled: 200",
    "time": "2015-07-13T19:19:48.387Z",
    "v": 0
}
```

* `304 Not Modified` - The resource has not been modified since the date indicated in the request header.

* `404 Not Found` - The resource was not found. The response body contains additional error information

* `410 Gone` - The resource has been deleted. The response body contains additional error information.




## <a href="http://www.vegapi.org" class="button">Return to Homepage</a>