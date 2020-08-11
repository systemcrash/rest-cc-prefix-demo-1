# REST CC (Call Control) DB

Use REST Call Control capabilities of your SBC together with a RESTful database to free your SBC of business logic. This example shows how to select a caller ID from the RESTful JSON database.

Keywords: REST, JSON, cURL, SBC

Required Firmware: v6.3.2

## Description

You run a call centre abroad, and your SIP trunk has some trunk phone numbers in other countries. You want to tidy up your number presentation. Presentation is, after all, everything. Right? Your call-centre has *inbound* numbers available to present as your CLI/DDI on a per-country basis when calling out from your global call-centre. You want to use a particular CLID or Caller Line ID (one that is available for use) on your SIP trunk. The callee number (the person to whom you are calling), will drive CLID selection. 

Summary: to whom you call sets your outbound CLID, the phone number you call from. This number is pulled live from an off-device database. 

## Detailed Overview 

Your SBC (remote to this repo and which has Internet access) captures a country-code (CC) prefix from an ingress Request-URI, and sends it in a concurrent cURL request to the endpoint hosting this database. In this example database there are three fields for each CC:

- `PAID` - holds the number to go into the `P-Asserted-Identity` header
- `From` - holds the number to go into the `From` header
- `Comment` 


So for `CC = 1`:

| CC      | 1 |
|---------|---|
| PAID    |13105552345|
| From    |13105552345|
| Comment |USA|


When the cURL request returns, the values in this example are supplied to GHM (General Header Manipulation) expressions which supplant headers into egress requests and responses with these provided values. In effect, we can, among other things, rewrite our CLID, with numbers taken from a remote database.

[Click here]( https://my-json-server.typicode.com/systemcrash/rest-cc-prefix-demo-1/prefix?cc=1 ) to see these example values for `CC = 1` returned by the RESTful database.

## Resources and dependencies

Firmware >= 6.3.2

[My JSON Server](https://my-json-server.typicode.com/) which describes itself as a `Fake Online REST server for teams`. Read more about it to understand how this demonstration works. For this demo, My JSON Server uses the `db.json` file in this repository.

On your Ingate SBC, under:
- __SIP Services, Basic Settings__: You must have a SIP TCP port configured (only UDP or TLS is not sufficient). It does not need to be `Active`.

Not required but a highly recommended guide if you do not understand Regular Expressions (regexp):

[Regular Expressions for Regular Folk](https://github.com/shreyasminocha/regex-for-regular-folk)


Note: hosting your own instance is a good option. TLS up to you. 

## Worked Example: Rewrite the `P-Asserted-Identity` header

John wants to call to the USA. But the SBC is not configured yet. So it's time to configure the SBC to use the RESTful database for number selection. This example is for the USA. 

---
Under the Call Control tab, add a REST API Server row:

| ... | ID  | Prefix | Suffix | ... |
|-----|------|-----|--------|-----|
|     | `1`  |`https://my-json-server.typicode.com/your-github-repo/rest-cc-prefix-demo-1/prefix?cc=`|        |     |

Whereby
|Path|Description|
|---|---|
|`your-github-repo/` |your github account/org name|
|`rest-cc-prefix-demo-1/`|this repo path|


You can even try with this repo - everything is ready: `https://my-json-server.typicode.com/systemcrash/rest-cc-prefix-demo-1/prefix?cc=`


Note: Later in the dial plan, the shortcut `$curl1` refers to the Server ID `1` above. To use `$curl2`, add a row with ID `2`, and so on.

Note: to adjust cURL timeout on v6.2.2 cumulatively patched, add (basic,advanced,options) advanced option name (sip.cc-timeout) and a value. Firmware >= 6.3.x has a dedicated field for this under REST CC.

---

In the dial plan, add a new:
* `Matching R-URI` row which captures a single digit prefix, or CC of 1 or 7 from the Request-URI:

| ... | Name | ... | Regexp | ... |
|-----|------|-----|--------|-----|
|     | `1x or 7x` |     | `sip:\+(1\|7)(.*)@.*` |     |

Note: `(1|7)` forms capture group `$1` or `$r1`. Capture group `$2` or `$r2` is `(.*)`.

* `Forward To` row which invokes our cURL expression:

| ... | Name | ... | Regexp | Trunk |...|
|-----|------|-----|--------|-----|---|
|  |`RESTCCPAID`| | `?P-Asserted-Identity=sip%3a$curl1($r1_XPATH//PAID/text())&!P-Bingo=sip%3a$curl1($r1_XPATH//PAID/text())`|`SIP Trunk 1`| |
|  |`RESTCCFrom`| | `?From=%3Csip%3a$curl1($r1_XPATH//From/text())%40$(ip.eth1)%3E&P-Asserted-Identity=sip%3a$curl1($r1_XPATH//PAID/text())&!P-Bingo=sip%3a$curl1($r1_XPATH//PAID/text())`|`SIP Trunk 1`| |

Note: `$r1` expresses the first capture group from our ingress Request-URI. In this example `$r1`, supplies the value `1` to the `$curl1(...)` expression.

* `Dial Plan` row: 

| ... | From | Request URI | Action | Forward To | ... |
|-----|------|-----|--------|-----|---|
|     | ...  | ... | ... | ... |
|     | ...  | `1x or 7x` | Forward | `RESTCCPAID` |


---

Our concurrent cURL request becomes [`https://my-json-server.typicode.com/systemcrash/rest-cc-prefix-demo-1/prefix?cc=1`]( https://my-json-server.typicode.com/systemcrash/rest-cc-prefix-demo-1/prefix?cc=1 ), to which the response comes 
```json
[
  {
    "cc": 1,
    "PAID": 13105552345,
    "From": 13105552345,
    "comment": "USA"
  }
]
```


Note: 

The Call Control function converts received JSON to XML internally, whereupon XPATH expressions are available. Our `$curl1` expression has `XPATH//PAID/text()`. This takes the `PAID` field text (which is a *number* field in the JSON above, but everything gets converted to text anyway). The `?P-Asserted-Identity=sip%3a...` GHM expression tells the B2BUA to insert/modify the `P-Asserted-Identity` header with the returned value `sip:13105552345`.


With everything configured and applied now, here is an ingress SIP INVITE from John:

```http
INVITE sip:+12125550000@192.0.2.5 SIP/2.0
Via: SIP/2.0/UDP 192.0.2.254:5060;branch=z9hG4bK-909121779;received=192.0.2.254;rport=5060
Content-Length: 0
From: "John" <sip:90501@192.0.2.254>;tag=633
Accept:
User-Agent: Johns-phone
To: "USA" <sip:12125550000@192.0.2.5>
Contact: sip:90501@192.0.2.254:5060
CSeq: 1 INVITE
Call-ID: 355593493830278013560573
Max-Forwards: 70
```

The Request-URI contains the user portion `+12125550000`. An INVITE that egresses to our operator might look like:

```http
INVITE sip:+12125550000@192.0.2.1 SIP/2.0
Via: SIP/2.0/UDP 192.0.2.5:5060;branch=z9hG4bK-909121779
Content-Length: 0
From: "John" <sip:90501@192.0.2.5>;tag=633
User-Agent: Franken-SBC
To: "USA" <sip:+12125550000@192.0.2.1>
P-Asserted-Identity: sip:13105552345
Contact: sip:zyx123@192.0.2.5
CSeq: 1 INVITE
Call-ID: 1
Max-Forwards: 70
```

## Worked Example: Rewrite the `From` Header
*Requires Firmware >=6.3.2 to fix a From header rewrite bug with REST Call Control*

Change your Dial plan to:

* `Dial Plan` row: 

| ... | From | Request URI | Action | Forward To | ... |
|-----|------|-----|--------|-----|---|
|     | ...  | ... | ... | ... |
|     | ...  | `1x or 7x` | Forward | `RESTCCFrom` |


And apply your configuration.


With everything configured and applied now, an INVITE that egresses to our operator might look like (note the modified `From` header):

```http
INVITE sip:+12125550000@192.0.2.1 SIP/2.0
Via: SIP/2.0/UDP 192.0.2.5:5060;branch=z9hG4bK-909121779
Content-Length: 0
From: "John" <sip:13105552345@192.0.2.5>;tag=633
User-Agent: Franken-SBC
To: "USA" <sip:+12125550000@192.0.2.1>
P-Asserted-Identity: sip:13105552345
Contact: sip:zyx123@192.0.2.5
CSeq: 1 INVITE
Call-ID: 1
Max-Forwards: 70
```


## Extra Points

The `RESTCCPAID` and `RESTCCFrom` GHM above contain: `...&!P-Bingo=sip%3a$curl1($r1_XPATH//PAID/text())` which instructs the B2BUA also to insert a `P-Bingo` header into all (1xx, 200 etc) INVITE responses. Not necessary, just possible, for your inspiration:

```http
SIP/2.0 180 Ringing
Via: SIP/2.0/UDP 192.0.2.5:5060;branch=z9hG4bK9d423e35
...
P-Bingo: sip:13105552345
...
```

## Code and Design (how to reproduce this JSON DB file)

Draw up your javascript DB at the following location

[https://beta5.objgen.com/json/local/design](https://beta5.objgen.com/json/local/design)

Clarification of the below JSON object design:

```json
prefix[]  <-- An array [], named 'prefix' which will hold all of the country codes.
  cc n = 1,  <-- Number field, country code.
  PAID n = 13105552345 <-- Number field, to insert into a P-Asserted-Identity header
  From n = 13105552345 <-- Number field, to insert into a From header
  comment = USA
```

Copy the below example JSON object code and paste at the above URL to produce formatted JSON output which will make our database file `db.json`, ready for my-json-server. 

```json

prefix[]
  cc n = 1, 
  PAID n = 13105552345
  From n = 13105552345
  comment = USA

prefix[]
  cc n = 7,
  PAID n = 74950123456
  From n = 74950123456
  comment = Russia

prefix[]
  cc n = 40
  PAID n = 40215552345
  From n = 40215552345
  comment = Romania

prefix[]
  cc n = 41
  PAID n = 414475552345
  From n = 414475552345
  comment = Switzerland

prefix[]
  cc n = 420
  PAID n = 420275552345
  From n = 420275552345
  comment = Czech Republic

prefix[]
  cc n = 421
  PAID n = 421275552345
  From n = 421275552345
  comment = Slovakia

prefix[]
  cc n = 423
  PAID n = 4230123456
  From n = 4230123456
  comment = Liechtenstein

prefix[]
  cc n = 43
  PAID n = 43175552345
  From n = 43175552345
  comment = Austria

prefix[]
  cc n = 44
  PAID n = 442075552345
  From n = 442075552345
  comment = UK

prefix[]
  cc n = 45
  PAID n = 4586007750
  From n = 4586007750
  comment = Denmark

prefix[]
  cc n = 46
  PAID n = 4686007750
  From n = 4686007750
  comment = Sweden

prefix[]
  cc n = 47
  PAID n = 4786007750
  From n = 4786007750
  comment = Norway

prefix[]
  cc n = 48
  PAID n = 48226007750
  From n = 48226007750
  comment = Poland

prefix[]
  cc n = 49
  PAID n = 49306007750
  From n = 49306007750
  comment = Germany

prefix[]
  cc n = 380
  PAID n= 380441234567
  From n= 380441234567
  comment = Ukraine

prefix[]
  cc n = 381
  PAID n = 3810912345678
  From n = 3810912345687
  comment = Serbia

prefix[]
  cc n = 382
  PAID n = 3820912345678
  From n = 3820912345678
  comment = Montenegro

prefix[]
  cc n = 383
  PAID n = 3830912345678
  From n = 3830912345678
  comment = Kosovo

prefix[]
  cc n = 385
  PAID n = 3850912345678
  From n = 3850912345678
  comment = Croatia

prefix[]
  cc n = 386
  PAID n = 3860912345678
  From n = 3860912345678
  comment = Slovenia

prefix[]
  cc n = 387
  PAID n = 3870912345678
  From n = 3870912345678
  comment = Bosnia and Herzegovina

profile
  name = typicode

```

---

The above JSON object model code should produce the following JSON:
```json

{
  "prefix": [
    {
      "cc": 1,
      "PAID": 13105552345,
      "From": 13105552345,
      "comment": "USA"
    },
    {
      "cc": 7,
      "PAID": 74950123456,
      "From": 74950123456,
      "comment": "Russia"
    },
    {
      "cc": 40,
      "PAID": 40215552345,
      "From": 40215552345,
      "comment": "Romania"
    },
    {
      "cc": 41,
      "PAID": 414475552345,
      "From": 414475552345,
      "comment": "Switzerland"
    },
    {
      "cc": 420,
      "PAID": 420275552345,
      "From": 420275552345,
      "comment": "Czech Republic"
    },
    {
      "cc": 421,
      "PAID": 421275552345,
      "From": 421275552345,
      "comment": "Slovakia"
    },
    {
      "cc": 423,
      "PAID": 4230123456,
      "From": 4230123456,
      "comment": "Liechtenstein"
    },
    {
      "cc": 43,
      "PAID": 43175552345,
      "From": 43175552345,
      "comment": "Austria"
    },
    {
      "cc": 44,
      "PAID": 442075552345,
      "From": 442075552345,
      "comment": "UK"
    },
    {
      "cc": 45,
      "PAID": 4586007750,
      "From": 4586007750,
      "comment": "Denmark"
    },
    {
      "cc": 46,
      "PAID": 4686007750,
      "From": 4686007750,
      "comment": "Sweden"
    },
    {
      "cc": 47,
      "PAID": 4786007750,
      "From": 4786007750,
      "comment": "Norway"
    },
    {
      "cc": 48,
      "PAID": 48226007750,
      "From": 48226007750,
      "comment": "Poland"
    },
    {
      "cc": 49,
      "PAID": 49306007750,
      "From": 49306007750,
      "comment": "Germany"
    },
    {
      "cc": 380,
      "PAID": 380441234567,
      "From": 380441234567,
      "comment": "Ukraine"
    },
    {
      "cc": 381,
      "PAID": 3810912345678,
      "From": 3810912345687,
      "comment": "Serbia"
    },
    {
      "cc": 382,
      "PAID": 3820912345678,
      "From": 3820912345678,
      "comment": "Montenegro"
    },
    {
      "cc": 383,
      "PAID": 3830912345678,
      "From": 3830912345678,
      "comment": "Kosovo"
    },
    {
      "cc": 385,
      "PAID": 3850912345678,
      "From": 3850912345678,
      "comment": "Croatia"
    },
    {
      "cc": 386,
      "PAID": 3860912345678,
      "From": 3860912345678,
      "comment": "Slovenia"
    },
    {
      "cc": 387,
      "PAID": 3870912345678,
      "From": 3870912345678,
      "comment": "Bosnia and Herzegovina"
    }
  ],
  "profile": {
    "name": "typicode"
  }
}

```
---


