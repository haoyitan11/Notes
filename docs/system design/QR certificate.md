# Payment Service Provider QR Certification System Design
## Maintenance Listing, Detail, API tester page
<img width="1784" height="781" alt="image" src="https://github.com/user-attachments/assets/b17f9489-0b77-4c5e-bb47-0fba2abd6fda" />

### Page design
HTML, Javascript

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

## Simulator
### Design
Java JDK21, Springboot 3.5.13, Maven 3.6.3

### Java dependencies
Jackson-databind (@JsonInclude)
Jackson-annotations (@JsonProperty)
commons-lang3 (StringUtils)
