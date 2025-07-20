# Salla Integration Guide with Flask
## Comprehensive Guide for Integrating Salla Platform with Flask Application to Retrieve User Information

---

## Table of Contents

1. [Overview](#overview)
2. [Prerequisites](#prerequisites)
3. [Application Setup](#application-setup)
4. [Webhooks Integration](#webhooks-integration)
5. [REST API Integration](#rest-api-integration)
6. [Practical Examples](#practical-examples)
7. [Best Practices](#best-practices)
8. [Troubleshooting](#troubleshooting)

---

## Overview

This guide enables you to create an effective integration with Salla platform to automatically retrieve user and order information when purchases occur. There are two main integration methods:

### 1. **Webhooks Integration (Recommended)**
- Receive instant notifications when specific events occur
- Ideal for automatic operations like user account creation

### 2. **REST API Integration**
- Direct data querying
- Ideal for retrieving additional details or data verification

---

## Prerequisites

### Required Software

```bash
# Create virtual environment
python -m venv salla_integration_env
source salla_integration_env/bin/activate  # Linux/Mac
# or
salla_integration_env\Scripts\activate     # Windows

# Install required libraries
pip install flask requests python-dotenv psycopg2-binary
```

### Salla Account and API Requirements

1. **Salla Store Account**: [https://salla.sa](https://salla.sa)
2. **Developer Application**: Create an app from the developer dashboard
3. **Required API Permissions**:
   - `orders.read` - Read orders
   - `customers.read` - Read customer data
   - `webhooks.read_write` - Manage webhooks

---

## Application Setup

### 1. Project Structure

```
salla_flask_integration/
├── app.py                 # Main Flask application
├── config.py             # Application settings
├── models.py             # Database models
├── salla_client.py       # Salla API client
├── webhook_handlers.py   # Webhook handlers
├── .env                  # Environment variables
└── requirements.txt      # Requirements
```

### 2. Environment File (.env)

```env
# Salla settings
SALLA_CLIENT_ID=your_client_id
SALLA_CLIENT_SECRET=your_client_secret
SALLA_ACCESS_TOKEN=your_access_token
SALLA_WEBHOOK_SECRET=your_webhook_secret

# Database settings
DATABASE_URL=postgresql://username:password@localhost:5432/salla_integration
# Or separate variables:
DB_HOST=localhost
DB_PORT=5432
DB_NAME=salla_integration
DB_USER=your_db_user
DB_PASSWORD=your_db_password

# Flask settings
FLASK_ENV=development
SECRET_KEY=your_secret_key_here
```

### 3. Application Configuration (config.py)

```python
import os
from dotenv import load_dotenv

load_dotenv()

class Config:
    """Basic application configuration"""
    
    # Flask settings
    SECRET_KEY = os.environ.get('SECRET_KEY') or 'dev-secret-key'
    
    # Salla settings
    SALLA_CLIENT_ID = os.environ.get('SALLA_CLIENT_ID')
    SALLA_CLIENT_SECRET = os.environ.get('SALLA_CLIENT_SECRET')
    SALLA_ACCESS_TOKEN = os.environ.get('SALLA_ACCESS_TOKEN')
    SALLA_WEBHOOK_SECRET = os.environ.get('SALLA_WEBHOOK_SECRET')
    SALLA_API_BASE_URL = 'https://api.salla.dev/admin/v2'
    
    # Database settings
    DATABASE_URL = os.environ.get('DATABASE_URL')
    
    @staticmethod
    def validate_config():
        """Verify all required settings are present"""
        required_vars = [
            'SALLA_CLIENT_ID',
            'SALLA_CLIENT_SECRET', 
            'SALLA_ACCESS_TOKEN'
        ]
        
        missing_vars = []
        for var in required_vars:
            if not os.environ.get(var):
                missing_vars.append(var)
        
        if missing_vars:
            raise ValueError(f"The following variables are required: {', '.join(missing_vars)}")
```

---

## Webhooks Integration

### 1. Webhook Handler Setup (webhook_handlers.py)

```python
import hashlib
import hmac
import json
from flask import current_app, request
from typing import Dict, Any, Optional

class WebhookVerifier:
    """Class to verify incoming webhooks from Salla"""
    
    @staticmethod
    def verify_signature(payload: bytes, signature: str, secret: str) -> bool:
        """
        Verify webhook signature to ensure it's from Salla
        
        Args:
            payload: Request content
            signature: Signature from Salla  
            secret: Shared secret key
            
        Returns:
            bool: True if signature is valid
        """
        try:
            expected_signature = hmac.new(
                secret.encode('utf-8'),
                payload,
                hashlib.sha256
            ).hexdigest()
            
            # Remove prefix if present
            if signature.startswith('sha256='):
                signature = signature[7:]
                
            return hmac.compare_digest(expected_signature, signature)
        except Exception as e:
            current_app.logger.error(f"Error verifying signature: {e}")
            return False

class WebhookProcessor:
    """Salla webhook event processor"""
    
    def __init__(self, user_service, salla_client):
        self.user_service = user_service
        self.salla_client = salla_client
    
    def process_event(self, event_type: str, data: Dict[str, Any]) -> Dict[str, Any]:
        """
        Process webhook events by type
        
        Args:
            event_type: Event type from Salla
            data: Event data
            
        Returns:
            Dict: Processing result
        """
        handlers = {
            'order.created': self._handle_order_created,
            'order.updated': self._handle_order_updated,
            'customer.created': self._handle_customer_created,
            'customer.updated': self._handle_customer_updated,
        }
        
        handler = handlers.get(event_type)
        if not handler:
            current_app.logger.warning(f"No handler for event: {event_type}")
            return {'status': 'ignored', 'message': 'Event type not supported'}
        
        try:
            return handler(data)
        except Exception as e:
            current_app.logger.error(f"Error processing event {event_type}: {e}")
            return {'status': 'error', 'message': str(e)}
    
    def _handle_order_created(self, data: Dict[str, Any]) -> Dict[str, Any]:
        """Handle new order created event"""
        order_id = data.get('id')
        customer_data = data.get('customer', {})
        
        current_app.logger.info(f"New order created: {order_id}")
        
        # Get additional details from API if needed
        order_details = self.salla_client.get_order_details(order_id)
        if not order_details:
            return {'status': 'error', 'message': 'Failed to fetch order details'}
        
        # Create or update user account
        user_result = self._create_or_update_user(customer_data, order_details)
        
        # Log order information
        self._log_order_info(order_details)
        
        return {
            'status': 'success',
            'message': 'Order processed successfully',
            'user_id': user_result.get('user_id'),
            'order_id': order_id
        }
    
    def _handle_customer_created(self, data: Dict[str, Any]) -> Dict[str, Any]:
        """Handle new customer created event"""
        customer_id = data.get('id')
        
        current_app.logger.info(f"New customer created: {customer_id}")
        
        # Create user account
        user_result = self._create_or_update_user(data)
        
        return {
            'status': 'success',
            'message': 'User account created successfully',
            'user_id': user_result.get('user_id'),
            'customer_id': customer_id
        }
    
    def _create_or_update_user(self, customer_data: Dict[str, Any], 
                             order_data: Optional[Dict[str, Any]] = None) -> Dict[str, Any]:
        """Create or update user account"""
        
        # Extract basic data
        salla_customer_id = customer_data.get('id')
        first_name = customer_data.get('first_name', '')
        last_name = customer_data.get('last_name', '')
        email = customer_data.get('email', '')
        mobile = customer_data.get('mobile', '')
        mobile_code = customer_data.get('mobile_code', '')
        
        # Compose full phone number
        phone = f"{mobile_code}{mobile}" if mobile_code and mobile else mobile
        
        # Create or update user in system
        user_data = {
            'salla_customer_id': salla_customer_id,
            'first_name': first_name,
            'last_name': last_name,
            'full_name': f"{first_name} {last_name}".strip(),
            'email': email,
            'phone': phone,
            'country': customer_data.get('country', ''),
            'city': customer_data.get('city', ''),
            'currency': customer_data.get('currency', 'SAR'),
        }
        
        # Add order information if available
        if order_data:
            user_data.update({
                'last_order_id': order_data.get('id'),
                'last_order_amount': order_data.get('amounts', {}).get('total', {}).get('amount', 0),
                'last_purchase_date': order_data.get('date', {}).get('date'),
            })
        
        # Save to database
        user_id = self.user_service.create_or_update_user(user_data)
        
        return {'user_id': user_id, 'data': user_data}
    
    def _log_order_info(self, order_data: Dict[str, Any]) -> None:
        """Log order information for review"""
        order_info = {
            'order_id': order_data.get('id'),
            'reference_id': order_data.get('reference_id'),
            'total_amount': order_data.get('amounts', {}).get('total', {}).get('amount'),
            'currency': order_data.get('currency'),
            'status': order_data.get('status', {}).get('name'),
            'payment_method': order_data.get('payment_method'),
            'items_count': len(order_data.get('items', [])),
            'date': order_data.get('date', {}).get('date')
        }
        
        current_app.logger.info(f"Order information: {json.dumps(order_info, ensure_ascii=False)}")
```

### 2. Salla API Client (salla_client.py)

```python
import requests
import json
from typing import Dict, Any, Optional, List
from flask import current_app

class SallaAPIClient:
    """Client for interacting with Salla API"""
    
    def __init__(self, config):
        self.base_url = config.SALLA_API_BASE_URL
        self.access_token = config.SALLA_ACCESS_TOKEN
        self.headers = {
            'Authorization': f'Bearer {self.access_token}',
            'Content-Type': 'application/json',
            'Accept': 'application/json'
        }
    
    def _make_request(self, method: str, endpoint: str, 
                     data: Optional[Dict] = None, 
                     params: Optional[Dict] = None) -> Optional[Dict[str, Any]]:
        """Make HTTP request to Salla API"""
        
        url = f"{self.base_url}{endpoint}"
        
        try:
            response = requests.request(
                method=method,
                url=url,
                headers=self.headers,
                json=data,
                params=params,
                timeout=30
            )
            
            if response.status_code == 200:
                return response.json()
            else:
                current_app.logger.error(
                    f"Salla API error - Code: {response.status_code}, "
                    f"Message: {response.text}"
                )
                return None
                
        except requests.exceptions.RequestException as e:
            current_app.logger.error(f"Connection error with Salla API: {e}")
            return None
    
    def get_order_details(self, order_id: int) -> Optional[Dict[str, Any]]:
        """Get specific order details"""
        endpoint = f"/orders/{order_id}"
        result = self._make_request('GET', endpoint)
        
        if result and result.get('success'):
            return result.get('data')
        return None
    
    def get_customer_details(self, customer_id: int) -> Optional[Dict[str, Any]]:
        """Get specific customer details"""
        endpoint = f"/customers/{customer_id}"
        result = self._make_request('GET', endpoint)
        
        if result and result.get('success'):
            return result.get('data')
        return None
    
    def list_orders(self, filters: Optional[Dict] = None) -> List[Dict[str, Any]]:
        """Fetch orders list with filters"""
        endpoint = "/orders"
        result = self._make_request('GET', endpoint, params=filters)
        
        if result and result.get('success'):
            return result.get('data', [])
        return []
    
    def register_webhook(self, webhook_data: Dict[str, Any]) -> Optional[Dict[str, Any]]:
        """Register new webhook"""
        endpoint = "/webhooks/subscribe"
        result = self._make_request('POST', endpoint, data=webhook_data)
        
        if result and result.get('success'):
            return result.get('data')
        return None
    
    def list_webhooks(self) -> List[Dict[str, Any]]:
        """Fetch registered webhooks list"""
        endpoint = "/webhooks"
        result = self._make_request('GET', endpoint)
        
        if result and result.get('success'):
            return result.get('data', [])
        return []
```

### 3. Database Models (models.py)

```python
from dataclasses import dataclass
from typing import Optional, List
import psycopg2
from psycopg2.extras import RealDictCursor
import json
from datetime import datetime
import os

@dataclass
class User:
    """User model"""
    id: Optional[int] = None
    salla_customer_id: Optional[int] = None
    first_name: str = ""
    last_name: str = ""
    full_name: str = ""
    email: str = ""
    phone: str = ""
    country: str = ""
    city: str = ""
    currency: str = "SAR"
    last_order_id: Optional[int] = None
    last_order_amount: float = 0.0
    last_purchase_date: Optional[str] = None
    created_at: Optional[str] = None
    updated_at: Optional[str] = None

class UserService:
    """User management service"""
    
    def __init__(self, db_url: str = None):
        # Parse DATABASE_URL or use individual components
        if db_url:
            self.db_url = db_url
        else:
            # Build connection string from individual components
            host = os.environ.get('DB_HOST', 'localhost')
            port = os.environ.get('DB_PORT', '5432')
            dbname = os.environ.get('DB_NAME', 'salla_integration')
            user = os.environ.get('DB_USER', 'postgres')
            password = os.environ.get('DB_PASSWORD', '')
            self.db_url = f"postgresql://{user}:{password}@{host}:{port}/{dbname}"
        
        self._init_database()
    
    def _get_connection(self):
        """Get database connection"""
        return psycopg2.connect(self.db_url)
    
    def _init_database(self):
        """Create database tables"""
        conn = self._get_connection()
        try:
            with conn.cursor() as cursor:
                # Create users table
                cursor.execute('''
                    CREATE TABLE IF NOT EXISTS users (
                        id SERIAL PRIMARY KEY,
                        salla_customer_id INTEGER UNIQUE,
                        first_name VARCHAR(255) NOT NULL,
                        last_name VARCHAR(255) NOT NULL,
                        full_name VARCHAR(255) NOT NULL,
                        email VARCHAR(255),
                        phone VARCHAR(50),
                        country VARCHAR(100),
                        city VARCHAR(100),
                        currency VARCHAR(10) DEFAULT 'SAR',
                        last_order_id INTEGER,
                        last_order_amount DECIMAL(10, 2) DEFAULT 0.0,
                        last_purchase_date TIMESTAMP,
                        created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
                        updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
                    )
                ''')
                
                # Create indexes for performance optimization
                cursor.execute('CREATE INDEX IF NOT EXISTS idx_salla_customer_id ON users(salla_customer_id)')
                cursor.execute('CREATE INDEX IF NOT EXISTS idx_email ON users(email)')
                cursor.execute('CREATE INDEX IF NOT EXISTS idx_phone ON users(phone)')
                
                # Create trigger to update updated_at timestamp
                cursor.execute('''
                    CREATE OR REPLACE FUNCTION update_updated_at_column()
                    RETURNS TRIGGER AS $$
                    BEGIN
                        NEW.updated_at = CURRENT_TIMESTAMP;
                        RETURN NEW;
                    END;
                    $$ language 'plpgsql';
                ''')
                
                cursor.execute('''
                    DROP TRIGGER IF EXISTS update_users_updated_at ON users;
                    CREATE TRIGGER update_users_updated_at BEFORE UPDATE ON users
                    FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();
                ''')
                
                conn.commit()
        finally:
            conn.close()
    
    def create_or_update_user(self, user_data: dict) -> int:
        """Create or update user"""
        
        conn = self._get_connection()
        try:
            with conn.cursor() as cursor:
                # Search for existing user
                cursor.execute(
                    'SELECT id FROM users WHERE salla_customer_id = %s',
                    (user_data.get('salla_customer_id'),)
                )
                existing_user = cursor.fetchone()
                
                if existing_user:
                    # Update existing user
                    user_id = existing_user[0]
                    
                    update_query = '''
                        UPDATE users SET 
                            first_name = %s, last_name = %s, full_name = %s,
                            email = %s, phone = %s, country = %s, city = %s, currency = %s,
                            last_order_id = %s, last_order_amount = %s, last_purchase_date = %s
                        WHERE id = %s
                        RETURNING id
                    '''
                    
                    cursor.execute(update_query, (
                        user_data.get('first_name', ''),
                        user_data.get('last_name', ''),
                        user_data.get('full_name', ''),
                        user_data.get('email', ''),
                        user_data.get('phone', ''),
                        user_data.get('country', ''),
                        user_data.get('city', ''),
                        user_data.get('currency', 'SAR'),
                        user_data.get('last_order_id'),
                        user_data.get('last_order_amount', 0.0),
                        user_data.get('last_purchase_date'),
                        user_id
                    ))
                    
                else:
                    # Create new user
                    insert_query = '''
                        INSERT INTO users (
                            salla_customer_id, first_name, last_name, full_name,
                            email, phone, country, city, currency,
                            last_order_id, last_order_amount, last_purchase_date
                        ) VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
                        RETURNING id
                    '''
                    
                    cursor.execute(insert_query, (
                        user_data.get('salla_customer_id'),
                        user_data.get('first_name', ''),
                        user_data.get('last_name', ''),
                        user_data.get('full_name', ''),
                        user_data.get('email', ''),
                        user_data.get('phone', ''),
                        user_data.get('country', ''),
                        user_data.get('city', ''),
                        user_data.get('currency', 'SAR'),
                        user_data.get('last_order_id'),
                        user_data.get('last_order_amount', 0.0),
                        user_data.get('last_purchase_date')
                    ))
                    
                    user_id = cursor.fetchone()[0]
                
                conn.commit()
                return user_id
        finally:
            conn.close()
    
    def get_user_by_salla_id(self, salla_customer_id: int) -> Optional[User]:
        """Find user by Salla ID"""
        conn = self._get_connection()
        try:
            with conn.cursor(cursor_factory=RealDictCursor) as cursor:
                cursor.execute('SELECT * FROM users WHERE salla_customer_id = %s', (salla_customer_id,))
                row = cursor.fetchone()
                
                if row:
                    return User(**row)
                return None
        finally:
            conn.close()
    
    def get_all_users(self, limit: int = 100, offset: int = 0) -> List[User]:
        """Fetch all users with pagination"""
        conn = self._get_connection()
        try:
            with conn.cursor(cursor_factory=RealDictCursor) as cursor:
                cursor.execute(
                    'SELECT * FROM users ORDER BY created_at DESC LIMIT %s OFFSET %s',
                    (limit, offset)
                )
                rows = cursor.fetchall()
                
                return [User(**row) for row in rows]
        finally:
            conn.close()
```

### 4. Main Application (app.py)

```python
from flask import Flask, request, jsonify, render_template_string
import json
import logging
from config import Config
from models import UserService
from salla_client import SallaAPIClient
from webhook_handlers import WebhookVerifier, WebhookProcessor
from datetime import datetime
import psycopg2
from psycopg2.extras import RealDictCursor
import os

# Application setup
app = Flask(__name__)
app.config.from_object(Config)

# Validate configuration
Config.validate_config()

# Setup logging
logging.basicConfig(level=logging.INFO)

# Create services with PostgreSQL connection
db_url = os.environ.get('DATABASE_URL')
user_service = UserService(db_url)
salla_client = SallaAPIClient(app.config)
webhook_processor = WebhookProcessor(user_service, salla_client)

@app.route('/')
def home():
    """Home page"""
    html_template = '''
    <!DOCTYPE html>
    <html dir="ltr" lang="en">
    <head>
        <meta charset="UTF-8">
        <meta name="viewport" content="width=device-width, initial-scale=1.0">
        <title>Salla Flask Integration</title>
        <style>
            body { font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif; margin: 40px; }
            .container { max-width: 800px; margin: 0 auto; }
            .card { background: #f8f9fa; padding: 20px; border-radius: 8px; margin: 20px 0; }
            .status { padding: 10px; border-radius: 4px; margin: 10px 0; }
            .success { background: #d4edda; color: #155724; border: 1px solid #c3e6cb; }
            .error { background: #f8d7da; color: #721c24; border: 1px solid #f5c6cb; }
            .info { background: #d1ecf1; color: #0c5460; border: 1px solid #bee5eb; }
            ul { padding-left: 20px; }
            .endpoint { background: #e9ecef; padding: 10px; border-radius: 4px; font-family: monospace; }
        </style>
    </head>
    <body>
        <div class="container">
            <h1>🛍️ Salla Flask Integration</h1>
            
            <div class="card">
                <h2>📊 System Status</h2>
                <div class="status success">✅ System running normally</div>
                <div class="status info">📡 Ready to receive webhooks from Salla</div>
            </div>
            
            <div class="card">
                <h2>🔗 Available Endpoints</h2>
                <ul>
                    <li><strong>Receive Webhooks:</strong> 
                        <div class="endpoint">POST /webhooks/salla</div>
                    </li>
                    <li><strong>List Users:</strong> 
                        <div class="endpoint">GET /users</div>
                    </li>
                    <li><strong>User Details:</strong> 
                        <div class="endpoint">GET /users/&lt;salla_customer_id&gt;</div>
                    </li>
                    <li><strong>Statistics:</strong> 
                        <div class="endpoint">GET /stats</div>
                    </li>
                </ul>
            </div>
            
            <div class="card">
                <h2>📝 Register Webhook in Salla</h2>
                <p>To register a webhook in Salla platform, use the following URL:</p>
                <div class="endpoint">{{ request.url_root }}webhooks/salla</div>
                <p><strong>Supported Events:</strong></p>
                <ul>
                    <li>order.created - New order created</li>
                    <li>order.updated - Order updated</li>
                    <li>customer.created - New customer created</li>
                    <li>customer.updated - Customer data updated</li>
                </ul>
            </div>
        </div>
    </body>
    </html>
    '''
    return render_template_string(html_template)

@app.route('/webhooks/salla', methods=['POST'])
def handle_salla_webhook():
    """Handle incoming webhooks from Salla"""
    
    try:
        # Verify Content-Type
        if not request.is_json:
            app.logger.error("Invalid webhook: Content-Type is not JSON")
            return jsonify({'error': 'Content-Type must be application/json'}), 400
        
        # Get data
        payload = request.get_data()
        signature = request.headers.get('X-Salla-Signature')
        
        if not signature:
            app.logger.error("Invalid webhook: No signature")
            return jsonify({'error': 'Webhook signature required'}), 400
        
        # Verify signature
        webhook_secret = app.config.get('SALLA_WEBHOOK_SECRET')
        if webhook_secret and not WebhookVerifier.verify_signature(payload, signature, webhook_secret):
            app.logger.error("Invalid webhook: Incorrect signature")
            return jsonify({'error': 'Invalid signature'}), 401
        
        # Process data
        data = request.get_json()
        event_type = data.get('event')
        event_data = data.get('data', {})
        
        app.logger.info(f"Received webhook from Salla: {event_type}")
        
        # Process event
        result = webhook_processor.process_event(event_type, event_data)
        
        return jsonify({
            'success': True,
            'message': 'Event processed successfully',
            'result': result
        }), 200
        
    except json.JSONDecodeError:
        app.logger.error("JSON parsing error")
        return jsonify({'error': 'Invalid JSON data'}), 400
    
    except Exception as e:
        app.logger.error(f"Error processing webhook: {e}")
        return jsonify({'error': 'Internal server error'}), 500

@app.route('/users')
def list_users():
    """Display users list"""
    
    try:
        page = int(request.args.get('page', 1))
        limit = int(request.args.get('limit', 10))
        offset = (page - 1) * limit
        
        users = user_service.get_all_users(limit=limit, offset=offset)
        
        # Convert users to dictionary
        users_data = []
        for user in users:
            users_data.append({
                'id': user.id,
                'salla_customer_id': user.salla_customer_id,
                'full_name': user.full_name,
                'email': user.email,
                'phone': user.phone,
                'country': user.country,
                'city': user.city,
                'currency': user.currency,
                'last_order_amount': user.last_order_amount,
                'last_purchase_date': user.last_purchase_date,
                'created_at': user.created_at
            })
        
        return jsonify({
            'success': True,
            'data': users_data,
            'pagination': {
                'page': page,
                'limit': limit,
                'total': len(users_data)
            }
        })
        
    except Exception as e:
        app.logger.error(f"Error fetching users: {e}")
        return jsonify({'error': 'Error fetching users'}), 500

@app.route('/users/<int:salla_customer_id>')
def get_user_details(salla_customer_id):
    """Display specific user details"""
    
    try:
        user = user_service.get_user_by_salla_id(salla_customer_id)
        
        if not user:
            return jsonify({'error': 'User not found'}), 404
        
        # Get additional details from Salla
        salla_customer = salla_client.get_customer_details(salla_customer_id)
        
        user_data = {
            'local_data': {
                'id': user.id,
                'salla_customer_id': user.salla_customer_id,
                'full_name': user.full_name,
                'email': user.email,
                'phone': user.phone,
                'country': user.country,
                'city': user.city,
                'currency': user.currency,
                'last_order_amount': user.last_order_amount,
                'last_purchase_date': user.last_purchase_date,
                'created_at': user.created_at,
                'updated_at': user.updated_at
            },
            'salla_data': salla_customer
        }
        
        return jsonify({
            'success': True,
            'data': user_data
        })
        
    except Exception as e:
        app.logger.error(f"Error fetching user details: {e}")
        return jsonify({'error': 'Error fetching user details'}), 500

@app.route('/stats')
def get_statistics():
    """Display system statistics"""
    
    try:
        # Statistics from PostgreSQL database
        conn = psycopg2.connect(os.environ.get('DATABASE_URL'))
        try:
            with conn.cursor() as cursor:
                # User count
                cursor.execute('SELECT COUNT(*) FROM users')
                total_users = cursor.fetchone()[0]
                
                # Total sales
                cursor.execute('SELECT SUM(last_order_amount) FROM users WHERE last_order_amount > 0')
                total_sales = cursor.fetchone()[0] or 0
                
                # New users today
                cursor.execute('''
                    SELECT COUNT(*) FROM users 
                    WHERE DATE(created_at) = CURRENT_DATE
                ''')
                new_users_today = cursor.fetchone()[0]
            
            stats = {
                'total_users': total_users,
                'total_sales': float(total_sales),
                'new_users_today': new_users_today,
                'currency': 'SAR',
                'last_updated': datetime.now().isoformat()
            }
            
            return jsonify({
                'success': True,
                'data': stats
            })
        finally:
            conn.close()
            
    except Exception as e:
        app.logger.error(f"Error fetching statistics: {e}")
        return jsonify({'error': 'Error fetching statistics'}), 500

@app.route('/webhooks/register', methods=['POST'])
def register_webhook():
    """Register new webhook in Salla"""
    
    try:
        webhook_url = f"{request.url_root}webhooks/salla"
        
        webhook_data = {
            "name": "Flask Salla Integration",
            "event": "order.created",
            "url": webhook_url,
            "version": 2
        }
        
        result = salla_client.register_webhook(webhook_data)
        
        if result:
            return jsonify({
                'success': True,
                'message': 'Webhook registered successfully',
                'data': result
            })
        else:
            return jsonify({
                'success': False,
                'message': 'Failed to register webhook'
            }), 400
            
    except Exception as e:
        app.logger.error(f"Error registering webhook: {e}")
        return jsonify({'error': 'Error registering webhook'}), 500

if __name__ == '__main__':
    app.run(debug=True, host='0.0.0.0', port=5000)
```

---

## Practical Examples

### 1. Running the Application

```bash
# Run the application
python app.py

# Application will run on
# http://localhost:5000
```

### 2. Register Webhook in Salla

```python
# Example to register webhook programmatically
import requests

webhook_data = {
    "name": "Flask Integration - Order Creation",
    "event": "order.created",
    "url": "https://your-domain.com/webhooks/salla",
    "version": 2,
    "headers": [
        {
            "key": "Authorization",
            "value": "Bearer your-secret-token"
        }
    ]
}

response = requests.post(
    "https://api.salla.dev/admin/v2/webhooks/subscribe",
    headers={
        "Authorization": "Bearer your-access-token",
        "Content-Type": "application/json"
    },
    json=webhook_data
)

print(response.json())
```

### 3. Example Incoming Webhook

```python
# Example data that will come from Salla when order is created
webhook_payload = {
    "event": "order.created",
    "data": {
        "id": 123456789,
        "reference_id": 1001,
        "status": {
            "id": 1,
            "name": "Pending Review",
            "slug": "under_review"
        },
        "customer": {
            "id": 987654321,
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
        },
        "date": {
            "date": "2024-01-15 14:30:00.000000",
            "timezone": "Asia/Riyadh"
        }
    }
}
```

---

## Best Practices

### 1. Security

```python
# Use environment variables for sensitive information
import os
from dotenv import load_dotenv

load_dotenv()

SALLA_ACCESS_TOKEN = os.getenv('SALLA_ACCESS_TOKEN')
WEBHOOK_SECRET = os.getenv('SALLA_WEBHOOK_SECRET')

# Always verify signature
def verify_webhook_signature(payload, signature, secret):
    import hmac
    import hashlib
    
    expected = hmac.new(
        secret.encode('utf-8'),
        payload,
        hashlib.sha256
    ).hexdigest()
    
    return hmac.compare_digest(expected, signature.replace('sha256=', ''))
```

### 2. Error Handling

```python
# Comprehensive error handling
import logging
from functools import wraps

def handle_exceptions(f):
    @wraps(f)
    def decorated_function(*args, **kwargs):
        try:
            return f(*args, **kwargs)
        except Exception as e:
            logging.error(f"Error in {f.__name__}: {e}")
            return {'error': 'Internal error'}, 500
    return decorated_function

@app.route('/api/endpoint')
@handle_exceptions
def my_endpoint():
    # Your code here
    pass
```

### 3. Logging

```python
# Setup comprehensive logging system
import logging
from logging.handlers import RotatingFileHandler

# Setup application log
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s %(levelname)s %(name)s %(message)s'
)

# Separate log for webhooks
webhook_logger = logging.getLogger('webhooks')
handler = RotatingFileHandler('webhooks.log', maxBytes=10000000, backupCount=5)
handler.setFormatter(logging.Formatter(
    '%(asctime)s %(levelname)s: %(message)s'
))
webhook_logger.addHandler(handler)
```

### 4. Data Validation

```python
from marshmallow import Schema, fields, validate

class OrderWebhookSchema(Schema):
    event = fields.Str(required=True, validate=validate.OneOf(['order.created', 'order.updated']))
    data = fields.Dict(required=True)

class CustomerSchema(Schema):
    id = fields.Int(required=True)
    first_name = fields.Str(required=True)
    last_name = fields.Str(required=True)
    email = fields.Email(allow_none=True)
    mobile = fields.Str(allow_none=True)

# Using validation
@app.route('/webhooks/salla', methods=['POST'])
def handle_webhook():
    schema = OrderWebhookSchema()
    try:
        result = schema.load(request.json)
        # Process validated data
    except ValidationError as err:
        return jsonify({'errors': err.messages}), 400
```

---

## Troubleshooting

### 1. Common Webhook Issues

| Issue | Possible Cause | Solution |
|-------|----------------|----------|
| Not receiving webhooks | Incorrect or unavailable URL | Ensure URL is working and accessible from internet |
| "Invalid signature" message | Wrong secret key | Verify WEBHOOK_SECRET is correct |
| Missing data | Missing fields in request | Check incoming data structure |

### 2. API Issues

```python
# Example API error handling
def handle_api_response(response):
    if response.status_code == 401:
        logging.error("Authentication error - Check Access Token")
        return None
    elif response.status_code == 429:
        logging.warning("Rate limit exceeded - Wait a moment")
        time.sleep(60)  # Wait one minute
        return None
    elif response.status_code >= 500:
        logging.error("Salla server error")
        return None
    
    return response.json()
```

### 3. Diagnostics

```python
# Diagnostic tool to check connection
@app.route('/debug/test-connection')
def test_salla_connection():
    try:
        # Test connection with Salla
        response = salla_client._make_request('GET', '/store/info')
        
        if response:
            return jsonify({
                'status': 'success',
                'message': 'Connection with Salla working normally',
                'store_info': response.get('data', {})
            })
        else:
            return jsonify({
                'status': 'error',
                'message': 'Failed to connect to Salla'
            }), 500
            
    except Exception as e:
        return jsonify({
            'status': 'error',
            'message': f'Connection error: {str(e)}'
        }), 500
```

---

## Conclusion

This guide provides a comprehensive framework for integrating with Salla platform using Flask. You can customize the code according to your specific needs and add more features such as:

- **Email notifications** when new accounts are created
- **Integration with other CRM systems**
- **Advanced analytics** for sales and customers
- **Admin interface** for monitoring activity

### Additional Tips for Success:

1. **Test integration** in development environment first
2. **Monitor logs** regularly to ensure system operation
3. **Keep backups** of the database
4. **Use HTTPS** in production environment
5. **Document changes** and updates

---

*This guide was created to help developers integrate smoothly with Salla platform. For technical support or inquiries, please refer to the official Salla documentation.*
