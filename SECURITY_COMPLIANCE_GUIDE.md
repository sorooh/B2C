# 🔒 GlobalMarket: Security & Compliance Guide
## Enterprise-Grade Security Architecture & Compliance Framework

**Version:** 1.0  
**Date:** November 28, 2025  
**Classification:** Internal - Confidential  
**Compliance:** GDPR, PCI DSS Level 1, ISO 27001  

---

## 📋 Table of Contents

1. [Security Overview](#security-overview)
2. [Authentication & Authorization](#authentication)
3. [Data Encryption](#encryption)
4. [GDPR Compliance](#gdpr)
5. [PCI DSS Compliance](#pci-dss)
6. [Security Monitoring](#monitoring)
7. [Incident Response](#incident-response)
8. [Penetration Testing](#penetration-testing)
9. [Security Checklist](#security-checklist)

---

<a name="security-overview"></a>
## 🛡️ 1. Security Overview

### 1.1 Security Principles

```yaml
Zero Trust Architecture:
  ✓ Never trust, always verify
  ✓ Least privilege access
  ✓ Assume breach mentality
  ✓ Continuous verification

Defense in Depth:
  ✓ Multiple security layers
  ✓ Redundant controls
  ✓ Fail-safe defaults
  ✓ Separation of duties

Security by Design:
  ✓ Secure coding practices
  ✓ Threat modeling
  ✓ Security reviews
  ✓ Automated testing
```

### 1.2 Security Layers

```
┌─────────────────────────────────────────────┐
│         Layer 7: User Education             │
├─────────────────────────────────────────────┤
│         Layer 6: Application Security       │
│   - Input validation, CSRF, XSS protection  │
├─────────────────────────────────────────────┤
│         Layer 5: API Security               │
│   - JWT tokens, Rate limiting, WAF          │
├─────────────────────────────────────────────┤
│         Layer 4: Network Security           │
│   - CloudFlare WAF, DDoS protection         │
├─────────────────────────────────────────────┤
│         Layer 3: Infrastructure Security    │
│   - Kubernetes RBAC, Network policies       │
├─────────────────────────────────────────────┤
│         Layer 2: Data Security              │
│   - Encryption at rest & in transit         │
├─────────────────────────────────────────────┤
│         Layer 1: Physical Security          │
│   - Azure datacenter security               │
└─────────────────────────────────────────────┘
```

### 1.3 Threat Model

```yaml
External Threats:
  - SQL Injection
  - Cross-Site Scripting (XSS)
  - Cross-Site Request Forgery (CSRF)
  - DDoS attacks
  - Credential stuffing
  - API abuse
  - Data breaches

Internal Threats:
  - Insider threats
  - Privilege escalation
  - Data exfiltration
  - Unauthorized access

Mitigation Strategies:
  - Input validation & sanitization
  - Parameterized queries
  - Content Security Policy
  - Rate limiting
  - Multi-factor authentication
  - Encryption everywhere
  - Audit logging
  - Regular security audits
```

---

<a name="authentication"></a>
## 🔐 2. Authentication & Authorization

### 2.1 Authentication Methods

```yaml
Primary Authentication:
  Method: JWT (JSON Web Tokens)
  Algorithm: RS256 (RSA with SHA-256)
  Access Token Lifetime: 15 minutes
  Refresh Token Lifetime: 30 days
  Token Storage: HttpOnly cookies (web) + Secure storage (mobile)

Multi-Factor Authentication (MFA):
  Enabled for: Vendors, Admins
  Optional for: Customers (recommended)
  Methods:
    - TOTP (Time-based One-Time Password)
    - SMS OTP
    - Email OTP
    - Authenticator apps (Google, Microsoft)

Password Requirements:
  - Minimum length: 12 characters
  - Must include: uppercase, lowercase, number, special char
  - Cannot be: common passwords, previous 5 passwords
  - Hashing: bcrypt (cost factor: 12)
  - Salt: Unique per user (32 bytes)
```

### 2.2 JWT Token Structure

```javascript
// Access Token
{
  "header": {
    "alg": "RS256",
    "typ": "JWT",
    "kid": "key-2025-11"
  },
  "payload": {
    "sub": "user_abc123",
    "tenant_id": "eu-west-1",
    "user_type": "customer",
    "email": "john.doe@example.com",
    "roles": ["customer"],
    "permissions": ["read:products", "write:orders"],
    "iat": 1701176400,
    "exp": 1701177300,
    "iss": "https://api.globalmarket.eu",
    "aud": "globalmarket-api"
  },
  "signature": "..."
}

// Refresh Token
{
  "payload": {
    "sub": "user_abc123",
    "type": "refresh",
    "jti": "token_unique_id_abc123",
    "iat": 1701176400,
    "exp": 1703768400  // 30 days
  }
}
```

### 2.3 Role-Based Access Control (RBAC)

```yaml
Roles & Permissions:

1. Customer:
   Permissions:
     - read:products
     - write:cart
     - write:orders
     - read:own_orders
     - write:reviews
     - read:own_profile
     - update:own_profile

2. Vendor:
   Permissions:
     - All customer permissions
     - write:products
     - read:own_products
     - update:own_products
     - delete:own_products
     - read:own_orders
     - update:order_status
     - read:analytics
     - read:payouts

3. Vendor Admin:
   Permissions:
     - All vendor permissions
     - manage:vendor_team
     - manage:vendor_settings
     - read:financial_reports

4. Platform Admin:
   Permissions:
     - read:all_data
     - write:all_data
     - manage:users
     - manage:vendors
     - manage:products (moderation)
     - manage:orders
     - read:system_logs
     - manage:platform_settings

5. Super Admin:
   Permissions:
     - All platform admin permissions
     - manage:admins
     - manage:security_settings
     - access:audit_logs
```

### 2.4 Authorization Implementation

```python
# decorators/permissions.py

from functools import wraps
from flask import request, jsonify

def require_auth(f):
    """Require valid JWT token"""
    @wraps(f)
    def decorated(*args, **kwargs):
        token = get_token_from_request()
        
        if not token:
            return jsonify({
                'error': 'AUTH_REQUIRED',
                'message': 'Authentication required'
            }), 401
        
        try:
            payload = verify_jwt_token(token)
            request.user = get_user_from_payload(payload)
            return f(*args, **kwargs)
        except TokenExpiredError:
            return jsonify({
                'error': 'TOKEN_EXPIRED',
                'message': 'Token has expired'
            }), 401
        except InvalidTokenError:
            return jsonify({
                'error': 'INVALID_TOKEN',
                'message': 'Invalid token'
            }), 401
    
    return decorated


def require_permission(*permissions):
    """Require specific permissions"""
    def decorator(f):
        @wraps(f)
        def decorated(*args, **kwargs):
            user = request.user
            
            if not user:
                return jsonify({
                    'error': 'AUTH_REQUIRED'
                }), 401
            
            user_permissions = get_user_permissions(user)
            
            if not all(p in user_permissions for p in permissions):
                return jsonify({
                    'error': 'PERMISSION_DENIED',
                    'message': 'Insufficient permissions',
                    'required': list(permissions),
                    'current': user_permissions
                }), 403
            
            return f(*args, **kwargs)
        
        return decorated
    return decorator


# Usage Example:
@app.route('/products', methods=['POST'])
@require_auth
@require_permission('write:products')
def create_product():
    # Only authenticated users with 'write:products' permission
    # can create products
    pass
```

---

<a name="encryption"></a>
## 🔐 3. Data Encryption

### 3.1 Encryption Strategy

```yaml
Encryption at Rest:
  Database:
    - Cosmos DB: Transparent Data Encryption (TDE) enabled
    - PostgreSQL: AES-256 encryption
    - Redis: Encryption enabled
  
  File Storage:
    - Azure Blob Storage: Server-side encryption (SSE)
    - Algorithm: AES-256
    - Key Management: Azure Key Vault
  
  Backups:
    - All backups encrypted
    - Separate encryption keys
    - Key rotation: Every 90 days

Encryption in Transit:
  - TLS 1.3 (minimum TLS 1.2)
  - Certificate: Let's Encrypt + CloudFlare
  - Perfect Forward Secrecy (PFS)
  - HSTS enabled (max-age: 31536000)
  - Certificate pinning for mobile apps

Field-Level Encryption:
  Sensitive Fields:
    - Passwords: bcrypt (cost: 12)
    - Credit card numbers: PCI DSS compliant tokenization
    - SSN/Tax ID: AES-256-GCM
    - Bank account numbers: AES-256-GCM
  
  Key Management:
    - Azure Key Vault
    - Key rotation: Every 90 days
    - Separate keys per tenant
```

### 3.2 Key Management

```python
# utils/encryption.py

from azure.keyvault.secrets import SecretClient
from azure.identity import DefaultAzureCredential
from cryptography.fernet import Fernet
import base64
import os

class EncryptionService:
    """Centralized encryption service"""
    
    def __init__(self):
        self.key_vault_url = os.getenv('KEY_VAULT_URL')
        self.credential = DefaultAzureCredential()
        self.client = SecretClient(
            vault_url=self.key_vault_url,
            credential=self.credential
        )
    
    def get_encryption_key(self, tenant_id: str) -> bytes:
        """Get tenant-specific encryption key"""
        secret_name = f"encryption-key-{tenant_id}"
        
        try:
            secret = self.client.get_secret(secret_name)
            return base64.b64decode(secret.value)
        except Exception:
            # Create new key if not exists
            return self.create_encryption_key(tenant_id)
    
    def create_encryption_key(self, tenant_id: str) -> bytes:
        """Create new encryption key for tenant"""
        key = Fernet.generate_key()
        secret_name = f"encryption-key-{tenant_id}"
        
        self.client.set_secret(
            secret_name,
            base64.b64encode(key).decode()
        )
        
        return key
    
    def encrypt_field(self, value: str, tenant_id: str) -> str:
        """Encrypt sensitive field"""
        key = self.get_encryption_key(tenant_id)
        fernet = Fernet(key)
        
        encrypted = fernet.encrypt(value.encode())
        return base64.b64encode(encrypted).decode()
    
    def decrypt_field(self, encrypted_value: str, tenant_id: str) -> str:
        """Decrypt sensitive field"""
        key = self.get_encryption_key(tenant_id)
        fernet = Fernet(key)
        
        encrypted_bytes = base64.b64decode(encrypted_value)
        decrypted = fernet.decrypt(encrypted_bytes)
        return decrypted.decode()


# Usage Example:
encryption_service = EncryptionService()

# Encrypt bank account
encrypted_account = encryption_service.encrypt_field(
    "NL91ABNA0417164300",
    tenant_id="eu-west-1"
)

# Decrypt when needed
account_number = encryption_service.decrypt_field(
    encrypted_account,
    tenant_id="eu-west-1"
)
```

---

<a name="gdpr"></a>
## 📋 4. GDPR Compliance

### 4.1 GDPR Requirements

```yaml
Legal Basis for Processing:
  - Consent: Marketing emails, cookies
  - Contract: Order processing, delivery
  - Legitimate Interest: Fraud prevention, analytics
  - Legal Obligation: Tax records, financial records

Data Subject Rights:
  1. Right to Access (Article 15)
  2. Right to Rectification (Article 16)
  3. Right to Erasure / "Right to be Forgotten" (Article 17)
  4. Right to Restriction of Processing (Article 18)
  5. Right to Data Portability (Article 20)
  6. Right to Object (Article 21)
  7. Rights related to Automated Decision Making (Article 22)

Data Protection Principles:
  - Lawfulness, fairness, transparency
  - Purpose limitation
  - Data minimization
  - Accuracy
  - Storage limitation
  - Integrity and confidentiality
  - Accountability

Breach Notification:
  - To Supervisory Authority: Within 72 hours
  - To Data Subjects: Without undue delay (if high risk)
```

### 4.2 Data Retention Policy

```yaml
Customer Data:
  Personal Information:
    - Active accounts: Indefinite (with consent)
    - Inactive accounts (>3 years): Anonymize or delete
    - Deleted accounts: 30-day grace period, then permanent deletion
  
  Order History:
    - Active orders: Indefinite
    - Completed orders: 7 years (legal requirement)
    - Cancelled orders: 2 years
  
  Financial Records:
    - Invoices: 7 years (legal requirement)
    - Payment records: 7 years
    - Refund records: 7 years

Vendor Data:
  Business Information: Duration of contract + 7 years
  Financial Records: 7 years after last transaction
  Product Data: Until vendor deletion + 30 days

Analytics & Logs:
  System Logs: 90 days
  Security Logs: 1 year
  Audit Logs: 7 years
  Analytics Data: 2 years (anonymized after 6 months)

Backup Data:
  Retention: 30 days
  Encrypted: Yes
  Deletion: Automatic after retention period
```

### 4.3 GDPR API Endpoints

```python
# api/gdpr.py

from flask import Blueprint, request, jsonify

gdpr_bp = Blueprint('gdpr', __name__)

@gdpr_bp.route('/data-export', methods=['POST'])
@require_auth
def export_user_data():
    """
    Article 15: Right to Access
    Article 20: Right to Data Portability
    """
    user = request.user
    
    # Collect all user data
    data = {
        'personal_info': get_user_profile(user.id),
        'orders': get_user_orders(user.id),
        'addresses': get_user_addresses(user.id),
        'payment_methods': get_payment_methods(user.id),
        'reviews': get_user_reviews(user.id),
        'wishlist': get_user_wishlist(user.id),
        'preferences': get_user_preferences(user.id)
    }
    
    # Create export file
    export_file = create_json_export(data)
    
    # Send email with download link
    send_export_email(user.email, export_file)
    
    # Log GDPR request
    log_gdpr_request(user.id, 'data_export')
    
    return jsonify({
        'success': True,
        'message': 'Data export will be sent to your email'
    })


@gdpr_bp.route('/delete-account', methods=['POST'])
@require_auth
def delete_user_account():
    """
    Article 17: Right to Erasure (Right to be Forgotten)
    """
    user = request.user
    password = request.json.get('password')
    
    # Verify password
    if not verify_password(user, password):
        return jsonify({
            'error': 'INVALID_PASSWORD'
        }), 401
    
    # Check if user has active orders
    active_orders = check_active_orders(user.id)
    if active_orders:
        return jsonify({
            'error': 'ACTIVE_ORDERS_EXIST',
            'message': 'Cannot delete account with active orders'
        }), 400
    
    # Schedule deletion (30-day grace period)
    schedule_account_deletion(user.id, days=30)
    
    # Send confirmation email
    send_deletion_confirmation_email(user.email)
    
    # Log GDPR request
    log_gdpr_request(user.id, 'account_deletion')
    
    return jsonify({
        'success': True,
        'message': 'Account will be deleted in 30 days',
        'cancellation_deadline': get_cancellation_deadline()
    })


@gdpr_bp.route('/consent', methods=['GET', 'POST'])
@require_auth
def manage_consent():
    """Manage user consent preferences"""
    user = request.user
    
    if request.method == 'GET':
        # Get current consent
        consent = get_user_consent(user.id)
        return jsonify({'data': consent})
    
    elif request.method == 'POST':
        # Update consent
        consent_data = request.json
        
        update_user_consent(user.id, {
            'marketing_emails': consent_data.get('marketing_emails'),
            'analytics': consent_data.get('analytics'),
            'personalization': consent_data.get('personalization')
        })
        
        # Log consent change
        log_consent_change(user.id, consent_data)
        
        return jsonify({
            'success': True,
            'message': 'Consent preferences updated'
        })
```

---

<a name="pci-dss"></a>
## 💳 5. PCI DSS Compliance

### 5.1 PCI DSS Requirements

```yaml
PCI DSS Level 1 Compliance:
  (Processing >6M transactions/year)

Build and Maintain Secure Network:
  Requirement 1: Install and maintain firewall configuration
  Requirement 2: Do not use vendor-supplied defaults

Protect Cardholder Data:
  Requirement 3: Protect stored cardholder data
  Requirement 4: Encrypt transmission of cardholder data

Maintain Vulnerability Management:
  Requirement 5: Protect against malware
  Requirement 6: Develop secure systems and applications

Implement Strong Access Control:
  Requirement 7: Restrict access to cardholder data
  Requirement 8: Identify and authenticate access
  Requirement 9: Restrict physical access

Monitor and Test Networks:
  Requirement 10: Track and monitor access
  Requirement 11: Regularly test security systems

Maintain Information Security Policy:
  Requirement 12: Maintain security policy
```

### 5.2 Payment Processing Strategy

```yaml
Strategy: PCI DSS Tokenization

Flow:
  1. Customer enters card → Frontend (SSL/TLS)
  2. Card data sent directly to Payment Gateway (Stripe/Mollie)
  3. Gateway returns token
  4. We store only token (NOT card data)
  5. Future charges use token

Compliance:
  ✓ We NEVER store card numbers
  ✓ We NEVER store CVV
  ✓ We NEVER store expiry dates
  ✓ All payment data handled by PCI-compliant gateway

Benefits:
  ✓ Reduced PCI DSS scope
  ✓ Lower security risk
  ✓ Easier compliance
  ✓ Gateway handles card data security
```

### 5.3 Payment Implementation

```javascript
// frontend/payment.js

async function processPayment(orderData) {
  // Step 1: Create payment intent on backend
  const response = await fetch('/api/payments/create-intent', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${accessToken}`
    },
    body: JSON.stringify({
      amount: orderData.total,
      currency: 'EUR',
      order_id: orderData.id
    })
  });
  
  const { client_secret } = await response.json();
  
  // Step 2: Use Stripe.js to collect card details
  // Card data NEVER touches our servers
  const stripe = Stripe('pk_live_...');
  
  const { error, paymentIntent } = await stripe.confirmCardPayment(
    client_secret,
    {
      payment_method: {
        card: cardElement,  // Stripe's secure element
        billing_details: {
          name: orderData.billing_name,
          email: orderData.email
        }
      }
    }
  );
  
  if (error) {
    // Payment failed
    handlePaymentError(error);
  } else if (paymentIntent.status === 'succeeded') {
    // Payment successful
    completeOrder(paymentIntent.id);
  }
}
```

---

<a name="monitoring"></a>
## 📊 6. Security Monitoring

### 6.1 Security Events to Monitor

```yaml
Authentication Events:
  - Failed login attempts (threshold: 5 in 5 minutes)
  - Successful login from new location
  - Password reset requests
  - MFA bypass attempts
  - Session hijacking attempts

Authorization Events:
  - Permission denied errors
  - Privilege escalation attempts
  - Unauthorized API access
  - Admin action logs

Data Access Events:
  - Bulk data exports
  - Sensitive data access
  - Database query patterns
  - Unusual API usage

Security Incidents:
  - SQL injection attempts
  - XSS attempts
  - CSRF token mismatches
  - Rate limit violations
  - DDoS attack patterns
  - Unusual traffic patterns

System Events:
  - Configuration changes
  - Deployment events
  - Certificate expiration warnings
  - Backup failures
```

### 6.2 Security Alerting

```python
# monitoring/security_alerts.py

import datadog
from datetime import datetime, timedelta

class SecurityAlertManager:
    """Manage security alerts and incidents"""
    
    def __init__(self):
        self.datadog_client = datadog.initialize()
    
    def check_failed_logins(self, user_id: str, ip_address: str):
        """Detect brute force attacks"""
        # Count failed logins in last 5 minutes
        failed_count = count_failed_logins(
            user_id=user_id,
            ip_address=ip_address,
            since=datetime.now() - timedelta(minutes=5)
        )
        
        if failed_count >= 5:
            # Block IP temporarily
            block_ip_address(ip_address, duration_minutes=30)
            
            # Send alert
            self.send_alert(
                severity='HIGH',
                event='BRUTE_FORCE_DETECTED',
                details={
                    'user_id': user_id,
                    'ip_address': ip_address,
                    'failed_attempts': failed_count
                }
            )
            
            # Log to SIEM
            log_security_event('brute_force', {
                'user_id': user_id,
                'ip': ip_address,
                'count': failed_count
            })
    
    def check_suspicious_api_usage(self, user_id: str):
        """Detect unusual API patterns"""
        # Check requests in last hour
        hourly_requests = count_api_requests(
            user_id=user_id,
            since=datetime.now() - timedelta(hours=1)
        )
        
        # Get user's normal pattern
        avg_requests = get_average_hourly_requests(user_id)
        
        if hourly_requests > avg_requests * 5:  # 5x normal
            self.send_alert(
                severity='MEDIUM',
                event='UNUSUAL_API_USAGE',
                details={
                    'user_id': user_id,
                    'hourly_requests': hourly_requests,
                    'average': avg_requests
                }
            )
    
    def send_alert(self, severity: str, event: str, details: dict):
        """Send security alert"""
        # Log to Datadog
        datadog.api.Event.create(
            title=f"Security Alert: {event}",
            text=f"Severity: {severity}\nDetails: {details}",
            alert_type='error' if severity == 'HIGH' else 'warning',
            tags=[f'security:{event}', f'severity:{severity}']
        )
        
        # Send to PagerDuty for HIGH severity
        if severity == 'HIGH':
            trigger_pagerduty_incident(event, details)
        
        # Send to Slack
        send_slack_notification(
            channel='#security-alerts',
            message=f"🚨 {severity}: {event}",
            details=details
        )
```

---

<a name="incident-response"></a>
## 🚨 7. Incident Response Plan

### 7.1 Incident Response Phases

```yaml
Phase 1: Preparation
  - Security team trained
  - Incident response plan documented
  - Tools and access ready
  - Communication channels established

Phase 2: Detection & Analysis
  - Security monitoring alerts
  - Identify incident type and severity
  - Determine scope and impact
  - Collect evidence

Phase 3: Containment
  - Short-term containment (isolate affected systems)
  - Long-term containment (apply patches, update rules)
  - Preserve evidence

Phase 4: Eradication
  - Remove threat
  - Patch vulnerabilities
  - Update security controls

Phase 5: Recovery
  - Restore systems
  - Verify security
  - Monitor for reoccurrence

Phase 6: Post-Incident
  - Document lessons learned
  - Update security controls
  - Improve monitoring
  - Train team
```

### 7.2 Incident Severity Levels

```yaml
CRITICAL (P1):
  - Data breach (customer/payment data)
  - Complete service outage
  - Active attack in progress
  Response Time: Immediate (< 15 minutes)
  Escalation: CEO, CTO, Security Team

HIGH (P2):
  - Partial data exposure
  - Major service degradation
  - Detected vulnerability exploitation
  Response Time: < 1 hour
  Escalation: CTO, Security Team

MEDIUM (P3):
  - Security policy violation
  - Minor service issues
  - Suspicious activity detected
  Response Time: < 4 hours
  Escalation: Security Team

LOW (P4):
  - Security best practice violation
  - No immediate risk
  Response Time: < 24 hours
  Escalation: Dev Team
```

---

<a name="security-checklist"></a>
## ✅ 9. Security Checklist

### 9.1 Pre-Launch Security Checklist

```yaml
Infrastructure:
  ☐ TLS 1.3 enabled
  ☐ HSTS configured
  ☐ CloudFlare WAF enabled
  ☐ DDoS protection active
  ☐ Firewall rules configured
  ☐ Network segmentation implemented
  ☐ VPN for admin access

Application:
  ☐ Input validation on all endpoints
  ☐ SQL injection protection (parameterized queries)
  ☐ XSS protection (Content Security Policy)
  ☐ CSRF protection enabled
  ☐ Rate limiting configured
  ☐ Session management secure
  ☐ Error messages don't leak info

Authentication:
  ☐ Strong password policy
  ☐ Password hashing (bcrypt)
  ☐ MFA available
  ☐ JWT tokens properly signed
  ☐ Token expiration configured
  ☐ Refresh token rotation

Data Protection:
  ☐ Encryption at rest (databases)
  ☐ Encryption in transit (TLS)
  ☐ Sensitive fields encrypted
  ☐ Key management (Azure Key Vault)
  ☐ Backup encryption
  ☐ No secrets in code

Compliance:
  ☐ GDPR compliance verified
  ☐ PCI DSS compliance (tokenization)
  ☐ Privacy policy published
  ☐ Cookie consent implemented
  ☐ Data retention policy
  ☐ Breach notification plan

Monitoring:
  ☐ Security logging enabled
  ☐ Audit trail for all actions
  ☐ Intrusion detection configured
  ☐ Anomaly detection active
  ☐ Alerts configured
  ☐ Incident response plan

Testing:
  ☐ Penetration testing completed
  ☐ Vulnerability scanning
  ☐ Security code review
  ☐ Dependency scanning
  ☐ OWASP Top 10 verified
```

---

**🎯 Security & Compliance Guide Complete!**

**التغطية الكاملة:**
- ✅ Authentication & Authorization (JWT, RBAC)
- ✅ Data Encryption (At rest, In transit, Field-level)
- ✅ GDPR Compliance (All 7 rights)
- ✅ PCI DSS (Tokenization strategy)
- ✅ Security Monitoring (Real-time alerts)
- ✅ Incident Response (6 phases)
- ✅ Security Checklist (Pre-launch)

**بكمل Infrastructure & DevOps Guide! 🚀**