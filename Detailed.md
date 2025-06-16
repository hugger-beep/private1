## Detailed System Architecture

```mermaid
graph TD
    subgraph "Client Layer"
        DIWebApp[DI WebApp]
    end
    
    subgraph "API Layer"
        APIGateway[API Gateway]
        APIAuth[API Authorizer Lambda]
    end
    
    subgraph "Authentication Layer"
        AuthMicro[Authentication Microservice Lambda]
    end
    
    subgraph "Reporting Layer"
        ReportMicro[Report Microservice]
        QS[QuickSight]
    end
    
    subgraph "Data Layer"
        RDSProxy[RDS Proxy]
        RDS[RDS Database]
        DW[DI Data Warehouse]
        
        subgraph "ETL Processes"
            TransETL[Transaction Data Glue ETL]
            MasterETL[Master Data Glue ETL]
            SecureSync[Secure Access Data Sync]
            ManualUpdate[Manual Exam Data Update]
        end
    end
    
    subgraph "External Systems"
        XACTDataAPI[Data APIs]
        XACIAuth[Authentication API]
    end
    
    DIWebApp --> APIGateway
    
    APIGateway --> AuthMicro
    APIGateway --> ReportMicro
    
    AuthMicro --> XACIAuth
    AuthMicro --> RDSProxy
    APIAuth --> RDSProxy
    
    ReportMicro --> QS
    ReportMicro --> APIAuth
    
    RDSProxy --> RDS
    RDS <--> SecureSync
    SecureSync --> DW
    TransETL --> DW
    MasterETL --> RDS
    ManualUpdate --> DW
    
    XACTDataAPI --> TransETL
    XACTDataAPI --> MasterETL
```

# Improved Multi-Region Architecture for customer

## Key Architecture Improvements

Based on the current architecture and requirements for global scaling with data residency compliance, the following improvements are recommended:

## 1. Multi-Region Architecture with Global Router

```mermaid
graph TD
    subgraph "Global Services"
        Route53[Route 53]
        GlobalDDB[(Organization Registry - DynamoDB Global Table)]
        CloudFront[CloudFront Distribution]
    end
    
    subgraph "Client Applications"
        MobileApp[Mobile Application]
        WebApp[Web Application]
    end
    
    subgraph "Global Router - ca-central-1"
        RouterAPI[Router API Gateway]
        RouterLambda[Router Lambda]
    end
    
    subgraph "Canada Region - ca-central-1"
        CAUserPool[Cognito User Pool - Canada]
        CAAPI[API Gateway - Canada]
        CAAuth[Authentication Microservice]
        CAReport[Report Microservice]
        CARDS[RDS - Canada]
        CADW[DI Data Warehouse - Canada]
        CASecureSync[Secure Access Data Sync]
        CAQS[QuickSight - Canada]
    end
    
    subgraph "US Region - us-east-1"
        USUserPool[Cognito User Pool - US]
        USAPI[API Gateway - US]
        USAuth[Authentication Microservice]
        USReport[Report Microservice]
        USRDS[RDS - US]
        USDW[DI Data Warehouse - US]
        USSecureSync[Secure Access Data Sync]
        USQS[QuickSight - US]
    end
    
    MobileApp --> Route53
    WebApp --> Route53
    Route53 --> CloudFront
    CloudFront --> RouterAPI
    
    RouterAPI --> RouterLambda
    RouterLambda --> GlobalDDB
    
    RouterLambda --> CAUserPool
    RouterLambda --> USUserPool
    
    CAUserPool --> CAAPI
    USUserPool --> USAPI
    
    CAAPI --> CAAuth
    CAAPI --> CAReport
    CAAuth --> CARDS
    CAReport --> CAQS
    CAReport --> CARDS
    CARDS --> CASecureSync
    CASecureSync --> CADW
    
    USAPI --> USAuth
    USAPI --> USReport
    USAuth --> USRDS
    USReport --> USQS
    USReport --> USRDS
    USRDS --> USSecureSync
    USSecureSync --> USDW
```

> **Note**:

## 2. Cognito-Based Authentication System

```mermaid
graph TD
    subgraph "Authentication Flow"
        User[User] --> AppAuth[App Authentication]
        AppAuth --> CognitoSDK[AWS Amplify SDK]
        CognitoSDK --> UserPool[Regional Cognito User Pool]
        UserPool --> MFA[Email MFA via Lambda Trigger]
        MFA --> JWT[JWT Token Issuance]
        JWT --> APIAccess[Regional API Access]
    end
    
    subgraph "User Synchronization"
        OrgTable[(Organization Registry - Global DynamoDB)]
        UserPool --> SyncLambda[User Sync Lambda]
        SyncLambda --> OtherRegions[Other Regional User Pools]
        OrgTable --> RouterLambda[Global Router Lambda]
    end
    
    subgraph "SSO Integration"
        EnterpriseIdP[Enterprise IdP]
        EnterpriseIdP --> SAML[SAML/OIDC Federation]
        SAML --> UserPool
    end
```

## Key Improvements

### 1. Global Routing Layer

**Current Issue:** No clear strategy for routing users to their region-specific resources.

**Improvement:**
- Implement a global router in a primary region (ca-central-1)
- Use Route 53 and CloudFront for global distribution
- Store organization-to-region mapping in DynamoDB Global Tables
- Router Lambda determines the correct region for each user/organization

**Benefits:**
- Single entry point for all users regardless of region
- Transparent redirection to region-specific resources
- Maintains data residency by routing to appropriate regional stack

**Organization-to-Region Mapping Example:**

The DynamoDB Global Table would contain entries like:

```json
{
  "organizationId": "org-123456",
  "organizationName": "Acme Corporation",
  "preferredRegion": "ca-central-1",
  "dataResidencyRequirement": "Canada",
  "createdAt": "2023-01-15T12:00:00Z",
  "updatedAt": "2023-01-15T12:00:00Z"
}

{
  "organizationId": "org-789012",
  "organizationName": "Global Enterprises",
  "preferredRegion": "us-east-1",
  "dataResidencyRequirement": "US",
  "createdAt": "2023-02-20T14:30:00Z",
  "updatedAt": "2023-03-10T09:15:00Z"
}
```

**Request Flow Explanation:**

1. User makes a request to your application domain (app.example.com)
2. Route 53 resolves the domain to the CloudFront distribution
3. CloudFront serves static content and forwards API requests to the Router API Gateway
4. Router Lambda looks up the user's organization in the Global DynamoDB table
5. Based on the organization's preferred region, the Router redirects to the appropriate regional stack

### 2. Cognito-Based Authentication

**Current Issue:** Custom authentication system lacks scalability and SSO capabilities.

**Improvement:**
- Replace custom Laravel authentication with Cognito User Pools in each region
- Implement custom Lambda triggers to maintain email-based MFA
- Use Cognito groups and attributes to store organization, business units, locations, and roles
- Enable SAML/OIDC federation for future SSO requirements

**Benefits:**
- Managed authentication service with built-in security features
- Support for future SSO requirements
- Consistent authentication experience across regions
- Reduced maintenance burden for authentication code

**Custom Email MFA Implementation:**

While Cognito does offer built-in MFA via SMS or TOTP (authenticator apps), it doesn't natively support email-based MFA. To maintain your current email MFA flow:

1. **Create a Custom Authentication Flow:**
   - Use Cognito's Custom Authentication Challenge Lambda Triggers
   - Implement three Lambda functions:
     - `DefineAuthChallenge`: Controls the authentication flow
     - `CreateAuthChallenge`: Generates and emails the MFA code
     - `VerifyAuthChallenge`: Verifies the user's response

2. **Email MFA Code Generation:**
```python
# CreateAuthChallenge Lambda
import json
import boto3
import random
import os

def lambda_handler(event, context):
    # Generate a random 6-digit code
    code = str(random.randint(100000, 999999))
    
    # Store the code in a secure way (with encryption)
    event['response'] = {
        'privateChallengeParameters': {'code': code}
    }
    
    # Get user email
    email = event['request']['userAttributes']['email']
    
    # Send email using SES or your existing email provider
    send_email_with_code(email, code)
    
    # Return the challenge to the user
    event['response']['publicChallengeParameters'] = {
        'email': email
    }
    
    return event

def send_email_with_code(email, code):
    # Example using AWS SES
    ses = boto3.client('ses')
    
    subject = "Your verification code"
    body = f"Your verification code is: {code}"
    
    ses.send_email(
        Source=os.environ['FROM_EMAIL'],
        Destination={'ToAddresses': [email]},
        Message={
            'Subject': {'Data': subject},
            'Body': {'Text': {'Data': body}}
        }
    )
```

**Cognito Attributes and Groups for Organization Structure:**

Cognito allows you to store custom attributes and assign users to groups, which can be used to model your organization structure:

1. **Custom Attributes:**
   - `custom:organizationId`: The organization the user belongs to
   - `custom:businessUnitIds`: Comma-separated list of business unit IDs
   - `custom:locationIds`: Comma-separated list of location IDs
   - `custom:userRole`: Role within the organization (e.g., "student", "employer")

2. **Cognito Groups:**
   - Create groups for major role categories (e.g., "Employers", "Students")
   - Groups can be used for coarse-grained access control

3. **Adding Claims to Tokens:**
   - Use a Pre-Token Generation Lambda trigger to add custom claims to the JWT tokens:

```python
# Pre-Token Generation Lambda
def lambda_handler(event, context):
    # Get user attributes
    user_attributes = event.get('userAttributes', {})
    
    # Add custom claims to the token
    event['response'] = {
        'claimsOverrideDetails': {
            'claimsToAddOrOverride': {
                'organization': user_attributes.get('custom:organizationId', ''),
                'businessUnits': user_attributes.get('custom:businessUnitIds', ''),
                'locations': user_attributes.get('custom:locationIds', ''),
                'role': user_attributes.get('custom:userRole', '')
            },
            'groupOverrideDetails': {
                'groupsToOverride': event['request']['groupConfiguration']['groupsToOverride'],
                'iamRolesToOverride': event['request']['groupConfiguration']['iamRolesToOverride'],
                'preferredRole': event['request']['groupConfiguration']['preferredRole']
            }
        }
    }
    
    return event
```

4. **Downstream Authorization:**
   - API Gateway Authorizers can extract and validate these claims
   - Your microservices can use these claims for fine-grained authorization
   - QuickSight can use these attributes for row-level security

### 3. Regional Data Isolation

**Current Issue:** No clear strategy for maintaining data residency requirements.

**Improvement:**
- Deploy complete application stacks in each region
- Maintain strict data isolation between regions
- Use Global DynamoDB table only for routing metadata, not customer data
- Regional QuickSight instances for reporting

**Benefits:**
- Clear data residency boundaries
- Compliance with regional data requirements
- Improved performance for users in each region

### 4. API Gateway Standardization

**Current Issue:** Inconsistent use of API Gateway across the application.

**Improvement:**
- Standardize on API Gateway for all API interactions
- Implement API Gateway Authorizers using Cognito
- Use regional API Gateways with consistent API definitions
- Implement API Gateway resource policies for additional security

**Benefits:**
- Consistent API management across regions
- Improved security with standardized authorization
- Better monitoring and throttling capabilities
- Simplified client integration

### 5. Enhanced ETL Processes

**Current Issue:** ETL processes need to be region-aware.

**Improvement:**
- Deploy ETL processes in each region
- Maintain region-specific data pipelines
- Use AWS Glue for all ETL processes
- Implement consistent tagging for regional resources

**Benefits:**
- Data processing remains within regional boundaries
- Simplified compliance with data residency requirements
- Consistent ETL approach across regions

## Implementation Roadmap

### Phase 1: Global Routing Infrastructure
1. Implement DynamoDB Global Table for organization registry
2. Develop Router Lambda function
3. Set up Route 53 and CloudFront distribution
4. Test routing logic with mock endpoints

### Phase 2: Cognito Implementation
1. Set up Cognito User Pools in each region
2. Develop custom Lambda triggers for email MFA
3. Implement user attribute synchronization between regions
4. Migrate users from existing authentication system

### Phase 3: Regional API Standardization
1. Define consistent API specifications
2. Implement regional API Gateways
3. Configure Cognito authorizers
4. Update client applications to use new authentication flow

### Phase 4: Data Layer Isolation
1. Ensure complete data isolation between regions
2. Implement regional ETL processes
3. Set up regional QuickSight instances
4. Validate data residency compliance

### Phase 5: SSO Integration
1. Configure SAML/OIDC federation in Cognito
2. Test with enterprise identity providers
3. Document SSO integration process for customers

## Security Considerations

1. **Token Handling**
   - Implement secure token storage in client applications
   - Configure appropriate token expiration times
   - Use refresh tokens for seamless user experience

2. **Cross-Region Access Control**
   - Strictly enforce region boundaries for data access
   - Implement additional checks in API authorizers
   - Log and alert on cross-region access attempts

3. **Monitoring and Auditing**
   - Set up CloudTrail logging for authentication events
   - Implement CloudWatch alarms for suspicious activities
   - Create dashboards for authentication metrics

4. **Data Protection**
   - Encrypt all data at rest and in transit
   - Implement least privilege access for all components
   - Regular security reviews and penetration testing
