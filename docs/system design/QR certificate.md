# Payment Service Provider QR Certification System Design
## Maintenance Listing, Detail, API tester page
<img width="1687" height="532" alt="image" src="https://github.com/user-attachments/assets/49a57611-b359-46d6-855d-818ea5745263" />

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
send http request to an endpoint that designed to execute Linux command (must fulfil Linux command)
