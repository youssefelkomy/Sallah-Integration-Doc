# Salla Integration Guide
## Deep Understanding of Salla Platform Integration Mechanism and How to Receive and Send Data

---

## Table of Contents

1. [Overview of the Working Mechanism](#overview)
2. [Integration Requirements](#integration-requirements)
3. [Understanding the Webhooks System](#understanding-webhooks)
4. [REST API Integration](#rest-api-integration)
5. [Data Schemas](#data-schemas)
6. [Practical Examples](#practical-examples)
7. [Error Handling and Security](#error-handling)

---

## Overview

Salla platform provides two main integration methods:

### 1. **Webhooks System (Preferred Method)**
- Receive instant notifications when certain events occur (new order, new customer)
- Suitable for automatic operations like user account creation
- **Principle**: Salla automatically sends data to your application

### 2. **REST API Integration**
- Direct query for data from Salla servers
- Suitable for getting additional details or verifying data
- **Principle**: Your application requests data from Salla when needed

---

## Integration Requirements

### Required Credentials

```json
{
  "client_id": "your_client_id",
  "client_secret": "your_client_secret", 
  "access_token": "your_access_token",
  "webhook_secret": "your_webhook_secret"
}
```

### Required Permissions (Scopes)
- `orders.read` - Read orders
- `customers.read` - Read customer data  
- `webhooks.read_write` - Manage webhooks

---

## Understanding Webhooks

### Working Mechanism
1. **Registration**: Register webhook URL in the control panel
2. **Reception**: Salla sends POST request when an event occurs
3. **Verification**: Verify request validity through signature verification
4. **Processing**: Extract and process data

### Example of Webhook Request from Salla

```http
POST /webhooks/salla HTTP/1.1
Host: your-domain.com
Content-Type: application/json
X-Salla-Signature: sha256=abc123...

{
  "event": "order.created",
  "data": {
    "id": 123456,
    "reference_id": 1001,
    "customer": {
      "id": 789,
      "first_name": "Ahmed",
      "last_name": "Mohammed",
      "email": "ahmed@example.com",
      "mobile": "555123456",
      "mobile_code": "+966"
    },
    "amounts": {
      "total": {
        "amount": 150.50,
        "currency": "SAR"
      }
    }
  }
}
```

### How to Verify Request Signature

```python
import hmac
import hashlib

def verify_webhook_signature(payload: bytes, signature: str, secret: str) -> bool:
    """Verify webhook signature validity"""
    
    # Create expected signature
    expected_signature = hmac.new(
        secret.encode('utf-8'),
        payload,
        hashlib.sha256
    ).hexdigest()
    
    # Remove prefix if present
    if signature.startswith('sha256='):
        signature = signature[7:]
    
    # Secure signature comparison
    return hmac.compare_digest(expected_signature, signature)

# Usage example
@app.route('/webhooks/salla', methods=['POST'])
def handle_webhook():
    payload = request.get_data()
    signature = request.headers.get('X-Salla-Signature')
    
    if not verify_webhook_signature(payload, signature, WEBHOOK_SECRET):
        return {'error': 'Invalid signature'}, 401
    
    # Process data
    data = request.get_json()
    process_webhook_event(data['event'], data['data'])
    
    return {'status': 'success'}, 200
```

---

## REST API Integration

### Base URL and Required Headers

```python
BASE_URL = "https://api.salla.dev/admin/v2"

HEADERS = {
    'Authorization': f'Bearer {ACCESS_TOKEN}',
    'Content-Type': 'application/json',
    'Accept': 'application/json'
}
```

### Key Endpoints

#### 1. Get Specific Order Details
```http
GET /orders/{order_id}
Authorization: Bearer your_access_token

Response:
{
  "success": true,
  "data": {
    "id": 123456,
    "reference_id": 1001,
    "status": {"name": "Under Review"},
    "customer": { ... },
    "items": [ ... ],
    "amounts": { ... }
  }
}
```

#### 2. Get Specific Customer Data
```http
GET /customers/{customer_id}
Authorization: Bearer your_access_token

Response:
{
  "success": true,
  "data": {
    "id": 789,
    "first_name": "Ahmed",
    "last_name": "Mohammed",
    "email": "ahmed@example.com",
    "mobile": "555123456",
    "mobile_code": "+966",
    "country": "Saudi Arabia",
    "city": "Riyadh"
  }
}
```

#### 3. Register New Webhook
```http
POST /webhooks/subscribe
Authorization: Bearer your_access_token
Content-Type: application/json

{
  "name": "Order Created Webhook",
  "event": "order.created", 
  "url": "https://your-domain.com/webhooks/salla",
  "version": 2
}

Response:
{
  "success": true,
  "data": {
    "id": 456,
    "name": "Order Created Webhook",
    "event": "order.created",
    "url": "https://your-domain.com/webhooks/salla"
  }
}
```

### Simple API Client Example

```python
import requests
import logging

class SallaAPIClient:
    def __init__(self, access_token: str):
        self.base_url = "https://api.salla.dev/admin/v2"
        self.headers = {
            'Authorization': f'Bearer {access_token}',
            'Content-Type': 'application/json'
        }
    
    def get_order(self, order_id: int) -> dict:
        """Get specific order details"""
        response = requests.get(
            f"{self.base_url}/orders/{order_id}",
            headers=self.headers
        )
        
        if response.status_code == 200:
            result = response.json()
            if result.get('success'):
                return result.get('data')
        
        logging.error(f"Failed to get order {order_id}: {response.text}")
        return None
    
    def get_customer(self, customer_id: int) -> dict:
        """Get specific customer data"""
        response = requests.get(
            f"{self.base_url}/customers/{customer_id}",
            headers=self.headers
        )
        
        if response.status_code == 200:
            result = response.json()
            if result.get('success'):
                return result.get('data')
        
        return None
```

---

## Data Schemas

### Webhook Events Schema

#### Order Created Event
```json
{
  "event": "order.created",
  "data": {
    "id": 123456,
    "reference_id": 1001,
    "status": {
      "id": 1,
      "name": "Under Review",
      "slug": "under_review"
    },
    "customer": {
      "id": 789,
      "first_name": "Ahmed", 
      "last_name": "Mohammed",
      "email": "ahmed@example.com",
      "mobile": "555123456",
      "mobile_code": "+966",
      "country": "Saudi Arabia",
      "city": "Riyadh"
    },
    "amounts": {
      "total": {
        "amount": 150.50,
        "currency": "SAR"
      },
      "tax": {
        "amount": 22.58,
        "currency": "SAR"
      }
    },
    "items": [
      {
        "id": 111,
        "name": "Test Product",
        "quantity": 2,
        "price": {
          "amount": 75.25,
          "currency": "SAR"
        }
      }
    ],
    "date": {
      "date": "2024-01-15 14:30:00.000000",
      "timezone": "Asia/Riyadh"
    }
  }
}
```

#### Customer Created Event
```json
{
  "event": "customer.created",
  "data": {
    "id": 789,
    "first_name": "Ahmed",
    "last_name": "Mohammed", 
    "email": "ahmed@example.com",
    "mobile": "555123456",
    "mobile_code": "+966",
    "country": "Saudi Arabia",
    "city": "Riyadh",
    "currency": "SAR",
    "date": {
      "date": "2024-01-15 14:30:00.000000",
      "timezone": "Asia/Riyadh"
    }
  }
}
```

### API Response Schema

#### Get Order Response
```json
{
  "success": true,
  "data": {
    "id": 123456,
    "reference_id": 1001,
    "status": {
      "id": 1,
      "name": "Under Review"
    },
    "customer": { ... },
    "shipping": {
      "company": "Shipping Company",
      "type": "Express Delivery",
      "cost": {
        "amount": 25.00,
        "currency": "SAR"
      }
    },
    "payment_method": "credit_card"
  }
}
```

### How to Extract Important Data

```python
def extract_user_data(webhook_data: dict) -> dict:
    """Extract user data from webhook"""
    
    customer = webhook_data.get('customer', {})
    order_data = webhook_data if 'amounts' in webhook_data else None
    
    # Basic data
    user_info = {
        'salla_customer_id': customer.get('id'),
        'first_name': customer.get('first_name', ''),
        'last_name': customer.get('last_name', ''),
        'email': customer.get('email', ''),
        'phone': f"{customer.get('mobile_code', '')}{customer.get('mobile', '')}",
        'country': customer.get('country', ''),
        'city': customer.get('city', ''),
        'currency': customer.get('currency', 'SAR')
    }
    
    # Order data (if available)
    if order_data:
        user_info.update({
            'last_order_id': order_data.get('id'),
            'last_order_amount': order_data.get('amounts', {}).get('total', {}).get('amount', 0),
            'last_purchase_date': order_data.get('date', {}).get('date')
        })
    
    return user_info
```

---

## Practical Examples

### 1. Setting Up a Simple Webhook Handler

```python
from flask import Flask, request, jsonify
import hmac
import hashlib

app = Flask(__name__)

@app.route('/webhooks/salla', methods=['POST'])
def handle_salla_webhook():
    """Simple handler for Salla webhooks"""
    
    # Verify signature
    payload = request.get_data()
    signature = request.headers.get('X-Salla-Signature', '').replace('sha256=', '')
    
    expected_signature = hmac.new(
        'your_webhook_secret'.encode(),
        payload,
        hashlib.sha256
    ).hexdigest()
    
    if not hmac.compare_digest(expected_signature, signature):
        return {'error': 'Invalid signature'}, 401
    
    # Process data
    data = request.get_json()
    event_type = data.get('event')
    event_data = data.get('data', {})
    
    if event_type == 'order.created':
        process_new_order(event_data)
    elif event_type == 'customer.created':
        process_new_customer(event_data)
    
    return {'status': 'success'}, 200

def process_new_order(order_data):
    """Process new order"""
    customer = order_data.get('customer', {})
    print(f"New order from customer: {customer.get('first_name')} {customer.get('last_name')}")
    
    # Save customer data to database
    save_customer_to_database(customer, order_data)

def process_new_customer(customer_data):
    """Process new customer"""
    print(f"New customer: {customer_data.get('first_name')} {customer_data.get('last_name')}")
    
    # Save customer data
    save_customer_to_database(customer_data)
```

### 2. Using API to Get Additional Data

```python
import requests

def get_order_details(order_id: int, access_token: str):
    """Get order details from Salla API"""
    
    headers = {
        'Authorization': f'Bearer {access_token}',
        'Content-Type': 'application/json'
    }
    
    response = requests.get(
        f'https://api.salla.dev/admin/v2/orders/{order_id}',
        headers=headers
    )
    
    if response.status_code == 200:
        result = response.json()
        if result.get('success'):
            return result.get('data')
    
    return None

# Usage example
order_details = get_order_details(123456, 'your_access_token')
if order_details:
    print(f"Order total: {order_details['amounts']['total']['amount']} SAR")
```

### 3. Register New Webhook

```python
def register_webhook(webhook_url: str, access_token: str):
    """Register webhook in Salla"""
    
    webhook_data = {
        "name": "Order Created Webhook",
        "event": "order.created",
        "url": webhook_url,
        "version": 2
    }
    
    headers = {
        'Authorization': f'Bearer {access_token}',
        'Content-Type': 'application/json'
    }
    
    response = requests.post(
        'https://api.salla.dev/admin/v2/webhooks/subscribe',
        headers=headers,
        json=webhook_data
    )
    
    if response.status_code == 200:
        result = response.json()
        if result.get('success'):
            print("Webhook registered successfully")
            return result.get('data')
    
    print("Failed to register webhook")
    return None

# Usage example
webhook_result = register_webhook(
    'https://your-domain.com/webhooks/salla',
    'your_access_token'
)
```

---

## Error Handling and Security

### 1. Data Validation

```python
def validate_webhook_data(data: dict) -> tuple[bool, str]:
    """Validate webhook data"""
    
    if not data.get('event'):
        return False, "event field is required"
    
    if not data.get('data'):
        return False, "data field is required"
    
    # Validate event type
    valid_events = ['order.created', 'order.updated', 'customer.created', 'customer.updated']
    if data['event'] not in valid_events:
        return False, f"Unsupported event type: {data['event']}"
    
    return True, "Data is valid"

# Usage example
@app.route('/webhooks/salla', methods=['POST'])
def handle_webhook():
    data = request.get_json()
    
    is_valid, message = validate_webhook_data(data)
    if not is_valid:
        return {'error': message}, 400
    
    # Process data
    process_webhook(data)
    return {'status': 'success'}, 200
```

### 2. API Error Handling

```python
import requests
from typing import Optional, Dict

def safe_api_request(url: str, headers: dict, timeout: int = 30) -> Optional[Dict]:
    """Safe API request with error handling"""
    
    try:
        response = requests.get(url, headers=headers, timeout=timeout)
        
        if response.status_code == 200:
            return response.json()
        elif response.status_code == 401:
            print("Authentication error - Check Access Token")
        elif response.status_code == 429:
            print("Rate limit exceeded - Wait a moment")
        elif response.status_code >= 500:
            print("Salla server error")
        else:
            print(f"Unexpected error: {response.status_code}")
            
    except requests.exceptions.Timeout:
        print("Request timeout")
    except requests.exceptions.ConnectionError:
        print("Connection error")
    except Exception as e:
        print(f"General error: {e}")
    
    return None
```

### 3. Security Best Practices

```python
import os
import logging
from functools import wraps

# Configure logging system
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s'
)

def secure_webhook_handler(f):
    """Decorator to secure webhook handler"""
    @wraps(f)
    def decorated_function(*args, **kwargs):
        # Verify Content-Type
        if not request.is_json:
            logging.warning("Webhook request with incorrect Content-Type")
            return {'error': 'Content-Type must be application/json'}, 400
        
        # Check for signature presence
        signature = request.headers.get('X-Salla-Signature')
        if not signature:
            logging.warning("Webhook request without signature")
            return {'error': 'Signature is required'}, 401
        
        # Verify signature validity
        payload = request.get_data()
        webhook_secret = os.getenv('SALLA_WEBHOOK_SECRET')
        
        if not verify_webhook_signature(payload, signature, webhook_secret):
            logging.error("Invalid webhook signature")
            return {'error': 'Invalid signature'}, 401
        
        return f(*args, **kwargs)
    
    return decorated_function

# Using the decorator
@app.route('/webhooks/salla', methods=['POST'])
@secure_webhook_handler
def handle_salla_webhook():
    data = request.get_json()
    # Process data securely
    return {'status': 'success'}, 200
```

---

## Integration Summary

### Important Points to Remember:

1. **Webhooks are the preferred method** for getting instant notifications
2. **Signature verification is essential** to ensure requests are actually from Salla
3. **Use API to get additional details** when needed
4. **Error handling is important** to ensure system stability
5. **Logging** helps track issues

### Ideal Workflow:
```
1. Salla sends webhook → Your app receives notification
2. Verify signature validity → Confirm request source  
3. Extract basic data → Process customer and order data
4. Request additional details from API → If necessary
5. Save data to database → Store customer information
6. Send success response to Salla → Confirm request receipt
```

This guide explains the basics required for proper and secure integration with Salla platform, focusing on understanding the working mechanism rather than creating a complete project.
