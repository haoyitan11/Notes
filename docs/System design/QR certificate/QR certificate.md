# Payment Service Provider QR Certification System Design
## Maintenance Listing, Detail, API tester page
<img width="1784" height="781" alt="image" src="https://github.com/user-attachments/assets/b17f9489-0b77-4c5e-bb47-0fba2abd6fda" />

<br><br>
### Page design
HTML, Javascript

<br><br>
### Data store
JSON format, .dat file
```json
{
  "test_case": "hy213",
  "test_description": "test",
  "test_objective": "test",
  "qr_type": "static",
  "services": [
    {
      "service": "qrquery",
      "npshost": "Y",
      "first_timeout": "N",
      "first_timeout_return_code": "",
      "second_timeout": "N",
      "second_timeout_return_code": "",
      "third_timeout": "N",
      "third_timeout_return_code": ""
    },
    {
      "service": "payadvice",
      "npshost": "Y",
      "first_timeout": "N",
      "first_timeout_return_code": "",
      "second_timeout": "N",
      "second_timeout_return_code": "",
      "third_timeout": "N",
      "third_timeout_return_code": ""
    },
    {
      "service": "payquery",
      "npshost": "Y",
      "first_timeout": "N",
      "first_timeout_return_code": "",
      "second_timeout": "N",
      "second_timeout_return_code": "",
      "third_timeout": "N",
      "third_timeout_return_code": ""
    }
  ],
  "advanced_options": {
    "qrquery-request": {
      "title": "QRQuery Request",
      "enabled": "N",
      "rows": []
    },
    "qrquery-response": {
      "title": "QRQuery Response",
      "enabled": "N",
      "rows": []
    },
    "payadvice-request": {
      "title": "PayAdvice Request",
      "enabled": "N",
      "rows": []
    },
    "payadvice-response": {
      "title": "PayAdvice Response",
      "enabled": "N",
      "rows": []
    },
    "payquery-request": {
      "title": "PayQuery Request",
      "enabled": "Y",
      "rows": [
        {
          "variable": "host_tid",
          "scenario": "same_as_payadvice_request",
          "value": ""
        }
      ]
    },
    "payquery-response": {
      "title": "PayQuery Response",
      "enabled": "Y",
      "rows": [
        {
          "variable": "response_code",
          "scenario": "same_as_payadvice_request",
          "value": ""
        }
      ]
    }
  }
}
```

### Data store
Send HTTP request – XMLHttpRequest dependencies
```javascript
    return new Promise((resolve) => {
        const xhr = new XMLHttpRequest();
        xhr.open("POST", destination, true);
        xhr.setRequestHeader("Content-Type", "application/json");
        xhr.setRequestHeader("KeyId", keyId);
        xhr.setRequestHeader("sign", signature);
        xhr.onreadystatechange = function () {
            if (xhr.readyState !== 4) return;
            resolve({
                ok: xhr.status === 200,
                status: xhr.status,
                headers: xhr.getAllResponseHeaders(),
                body: xhr.responseText,
                destination,
                signature,
                requestBody: payload,
            });
        };
        xhr.onerror = function () {
            resolve({ ok: false, status: 0, headers: "", body: "Network error", destination, signature, requestBody: payload });
        };
        xhr.send(payload);
    });
```

### Execute command
send http request to an endpoint that designed to execute window terminal command
```javascript
    run(cmd, isSilent = false) {
        let xh = new XMLHttpRequest();
        let apiPrefix = "/gopi";
        xh.open("GET", apiPrefix + "/ss/cli?rcmd=" + encodeURIComponent(cmd), false);
        xh.send(null);
        let rpts = xh.responseText.split("\n");
        if (!isSilent) {
            rpts.forEach(ll => this.logfn(ll));
        }
        return rpts;
    }
```
<br><br>
## Simulator
### Design
Java JDK21, Springboot 3.5.13, Maven 3.6.3

### Spring dependencies
#### spring-boot-starter-web
- Jackson Databind (@JsonInclude), Jackson Annotations (@JsonProperty)
 
#### spring-boot-starter-test
- JUnit 5 (@Test, Assertions), Mockito (@Mock, @InjectMocks)
 
#### spring-integration-core
- Spring Messaging (Message, MessageBuilder), @ServiceActivator, @MessagingGateway
 
### Java Utility Dependencies
- Apache Commons Lang3 (StringUtils), Lombok (@Getter, @Setter, @Builder)

<br><br>
## Meeting Sessions
### Meeting (14/8/2026)
1. 3 pages (Maintenance Page, Listing page)
2. Wallet provider can customize the setup, response timeout

<p> <img width="940" height="385" alt="image" src="https://github.com/user-attachments/assets/8ab09ff0-8a6b-4af7-bfb7-8e14c16c6b41" /> </p>
<p> <img width="940" height="418" alt="image" src="https://github.com/user-attachments/assets/2caaa46e-c9bd-470b-852e-754af1b71f89" /> </p>

### Meeting (26/8/2026)
- no database access, use .dat, .json
- dont take from server.log, instead paste the response code to simulator.log
- simulator.log naming by day, so batch job can retrieve by day
- SDLC, requirement gathering have to ensure all planning are practical to achieve, dont try to complete first, then try to solve it
- add advanced option at maintainance detail page so varialbe can set its value
- static = reuse value, no need regen QRcode, while dynamic, if want generate QR u need to create order request first
- try to understand the python script from Lay Lee
- do validation for the sandbox environment

### Meeting (3/9/2026)
1. Have to confirm whether Maintainence page & listing require password indicator or not, since these 2 page only can be accessed by internal, external only can use API tester page 
2. QR generated, since test case name is not same as previous length, whether QR code value accept max length or not (test case name length)
3. New button and multiple select to dowload .dat file into local device
4. NPS simulator & batch is separated, NPS simulator, maintenance page, lising page uat, batch is on local device (generate csv file)
5. Log message generated by NPS host
6. 1 log file stored for 1 wallet provider, identify with institution_code and SOF_uri
7. MTI + process code is to identify its payadvice request/response, qrquery request/response

### Meeting (24/9/2026)
1. How Simulator log works (API tester generate QR > wallet provider scan > call API to our simulator > simulator call to NPS host > NPS host return response > simulator generate log to track these response), other than that , when simulator(receive request from WP also need to log)
2. Wallet provider pass in wrong MTI and process code to simulator > just let it go through nps-host, nps-host will reject automatically since the sequence has to be QRquery > payadvice > payquery
3. Some test case scenario writing PayQuery Request not present, maintenance detail has to add dropdownlist detail, for PayQuery object, if detected request sent to simulator, automatically declare as fail
- When first row is variable, then second row and next row cant pick PayQuery request for dropdownlist
- After PayQuery request picked at dropdownlist, second row and next cant pick variable
4. There is 1 scenario, txn_identifer read from QR code string, to design this, allow maintenance detail to upload QR code and javascript read the QR code value and save at .dat file, when this scenario picked then simulator will read .dat file
5. Instuition code + SOF_URI = determine specific wallet provider, so instuition code dropdownlist scenario should be tied to wallet provider SOF_URI, while SOF_URI scenario dropdownlist add tied to wallet provider instuition code
6. There is scenario where OrderRequest can choose connect / not to connect NPS host, so maintenance detail add 1 more service row for OrderRequest, but default tick to connect NPS host
