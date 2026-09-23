# Payment Service Provider QR Certification System Design
## Maintenance Listing, Detail, API tester page
<img width="1687" height="532" alt="image" src="https://github.com/user-attachments/assets/49a57611-b359-46d6-855d-818ea5745263" />

### Page design
HTML, Javascript

### Data store
JSON format, .dat file
```json
{
  "test_case": "test0",
  "test_description": "12121",
  "test_objective": "1212121",
  "qr_type": "static",
  "services": [
    {
      "service": "qrquery",
      "npshost": "N",
      "first_timeout": "N",
      "first_timeout_return_code": "",
      "second_timeout": "N",
      "second_timeout_return_code": "",
      "third_timeout": "N",
      "third_timeout_return_code": ""
    },
    {
      "service": "payadvice",
      "npshost": "N",
      "first_timeout": "N",
      "first_timeout_return_code": "",
      "second_timeout": "N",
      "second_timeout_return_code": "",
      "third_timeout": "N",
      "third_timeout_return_code": ""
    },
    {
      "service": "payquery",
      "npshost": "N",
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
      "enabled": true,
      "rows": [
        {
          "variable": "retrieval_ref",
          "scenario": "is_present",
          "value": "21321312"
        },
        {
          "variable": "txn_identifier",
          "scenario": "is_present",
          "value": "test"
        }
      ]
    },
    "qrquery-response": {
      "title": "QRQuery Response",
      "enabled": false,
      "rows": []
    },
    "payadvice-request": {
      "title": "PayAdvice Request",
      "enabled": false,
      "rows": []
    },
    "payadvice-response": {
      "title": "PayAdvice Response",
      "enabled": false,
      "rows": []
    },
    "payquery-request": {
      "title": "PayQuery Request",
      "enabled": false,
      "rows": []
    },
    "payquery-response": {
      "title": "PayQuery Response",
      "enabled": false,
      "rows": []
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
send http request to an endpoint that designed to execute Linux command (must fulfil Linux command)
