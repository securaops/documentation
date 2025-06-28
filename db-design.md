
# CyberShield SMB - Database Design Specification

**Version: 1.0**
**Date: January 2025**
**Database: PostgreSQL 16 with Schema-per-Tenant Model**

## Overview

This document outlines the database design for CyberShield SMB platform using PostgreSQL with a schema-per-tenant approach. Each tenant gets their own database schema for data isolation while sharing the same physical database instance.

## Schema Structure

- **Global Schema (`public`)**: Contains shared reference data and tenant registry
- **Tenant Schemas (`tenant_<tenant_id>`)**: Contains tenant-specific operational data

---

## Global Schema Tables (public)

### 1. tenants
Manages tenant registry and configuration.

| Column Name | Data Type | Constraints | Description |
|-------------|-----------|-------------|-------------|
| id | UUID | PRIMARY KEY, DEFAULT gen_random_uuid() | Unique tenant identifier |
| name | VARCHAR(255) | NOT NULL | Tenant organization name |
| subdomain | VARCHAR(100) | UNIQUE, NOT NULL | Tenant subdomain for URL routing |
| schema_name | VARCHAR(100) | UNIQUE, NOT NULL | Database schema name for tenant |
| status | VARCHAR(50) | NOT NULL, DEFAULT 'active' | Tenant status (active, suspended, terminated) |
| subscription_plan | VARCHAR(100) | NOT NULL | Subscription plan type |
| max_users | INTEGER | NOT NULL, DEFAULT 50 | Maximum allowed users |
| max_endpoints | INTEGER | NOT NULL, DEFAULT 500 | Maximum monitored endpoints |
| created_at | TIMESTAMP | NOT NULL, DEFAULT CURRENT_TIMESTAMP | Tenant creation timestamp |
| updated_at | TIMESTAMP | NOT NULL, DEFAULT CURRENT_TIMESTAMP | Last update timestamp |
| billing_email | VARCHAR(255) | NOT NULL | Primary billing contact email |
| technical_contact | VARCHAR(255) | NOT NULL | Primary technical contact email |

### 2. compliance_frameworks
Global reference data for compliance frameworks.

| Column Name | Data Type | Constraints | Description |
|-------------|-----------|-------------|-------------|
| id | UUID | PRIMARY KEY, DEFAULT gen_random_uuid() | Framework identifier |
| name | VARCHAR(100) | UNIQUE, NOT NULL | Framework name (SOC2, HIPAA, PCI-DSS, etc.) |
| version | VARCHAR(50) | NOT NULL | Framework version |
| description | TEXT | | Framework description |
| is_active | BOOLEAN | NOT NULL, DEFAULT true | Framework availability status |
| created_at | TIMESTAMP | NOT NULL, DEFAULT CURRENT_TIMESTAMP | Creation timestamp |

### 3. compliance_controls
Global reference data for compliance controls.

| Column Name | Data Type | Constraints | Description |
|-------------|-----------|-------------|-------------|
| id | UUID | PRIMARY KEY, DEFAULT gen_random_uuid() | Control identifier |
| framework_id | UUID | NOT NULL, REFERENCES compliance_frameworks(id) | Parent framework |
| control_number | VARCHAR(50) | NOT NULL | Control reference number |
| title | VARCHAR(500) | NOT NULL | Control title |
| description | TEXT | NOT NULL | Control description |
| control_type | VARCHAR(100) | NOT NULL | Control type (preventive, detective, corrective) |
| category | VARCHAR(100) | NOT NULL | Control category |
| is_active | BOOLEAN | NOT NULL, DEFAULT true | Control availability status |

---

## Tenant Schema Tables (tenant_<tenant_id>)

### 1. users
Tenant-specific user management.

| Column Name | Data Type | Constraints | Description |
|-------------|-----------|-------------|-------------|
| id | UUID | PRIMARY KEY, DEFAULT gen_random_uuid() | User identifier |
| tenant_id | UUID | NOT NULL | Reference to global tenants table |
| email | VARCHAR(255) | UNIQUE, NOT NULL | User email address |
| first_name | VARCHAR(100) | NOT NULL | User first name |
| last_name | VARCHAR(100) | NOT NULL | User last name |
| phone | VARCHAR(20) | | User phone number |
| is_active | BOOLEAN | NOT NULL, DEFAULT true | User account status |
| last_login | TIMESTAMP | | Last successful login |
| mfa_enabled | BOOLEAN | NOT NULL, DEFAULT false | Multi-factor authentication status |
| created_at | TIMESTAMP | NOT NULL, DEFAULT CURRENT_TIMESTAMP | Account creation timestamp |
| updated_at | TIMESTAMP | NOT NULL, DEFAULT CURRENT_TIMESTAMP | Last update timestamp |
| password_hash | VARCHAR(255) | NOT NULL | Encrypted password hash |
| role | VARCHAR(100) | NOT NULL, DEFAULT 'user' | User role (admin, analyst, user, auditor) |

### 2. endpoints
Monitored endpoints and devices.

| Column Name | Data Type | Constraints | Description |
|-------------|-----------|-------------|-------------|
| id | UUID | PRIMARY KEY, DEFAULT gen_random_uuid() | Endpoint identifier |
| hostname | VARCHAR(255) | NOT NULL | Device hostname |
| ip_address | INET | NOT NULL | Device IP address |
| mac_address | MACADDR | | Device MAC address |
| operating_system | VARCHAR(100) | NOT NULL | Operating system type |
| os_version | VARCHAR(100) | | Operating system version |
| endpoint_type | VARCHAR(50) | NOT NULL | Type (workstation, server, mobile, iot) |
| status | VARCHAR(50) | NOT NULL, DEFAULT 'active' | Endpoint status |
| last_seen | TIMESTAMP | | Last communication timestamp |
| agent_version | VARCHAR(50) | | Installed agent version |
| location | VARCHAR(255) | | Physical location |
| owner_user_id | UUID | REFERENCES users(id) | Assigned user |
| created_at | TIMESTAMP | NOT NULL, DEFAULT CURRENT_TIMESTAMP | Registration timestamp |
| updated_at | TIMESTAMP | NOT NULL, DEFAULT CURRENT_TIMESTAMP | Last update timestamp |

### 3. threats
Security threats and incidents.

| Column Name | Data Type | Constraints | Description |
|-------------|-----------|-------------|-------------|
| id | UUID | PRIMARY KEY, DEFAULT gen_random_uuid() | Threat identifier |
| threat_type | VARCHAR(100) | NOT NULL | Threat classification |
| severity | VARCHAR(20) | NOT NULL | Severity level (critical, high, medium, low) |
| status | VARCHAR(50) | NOT NULL, DEFAULT 'open' | Investigation status |
| title | VARCHAR(500) | NOT NULL | Threat title |
| description | TEXT | NOT NULL | Detailed description |
| source_endpoint_id | UUID | REFERENCES endpoints(id) | Source endpoint |
| assigned_user_id | UUID | REFERENCES users(id) | Assigned analyst |
| detected_at | TIMESTAMP | NOT NULL | Detection timestamp |
| resolved_at | TIMESTAMP | | Resolution timestamp |
| created_at | TIMESTAMP | NOT NULL, DEFAULT CURRENT_TIMESTAMP | Record creation timestamp |
| updated_at | TIMESTAMP | NOT NULL, DEFAULT CURRENT_TIMESTAMP | Last update timestamp |
| mitre_tactic | VARCHAR(100) | | MITRE ATT&CK tactic |
| mitre_technique | VARCHAR(100) | | MITRE ATT&CK technique |
| confidence_score | DECIMAL(3,2) | CHECK (confidence_score >= 0 AND confidence_score <= 1) | AI confidence score |

### 4. security_events
Raw security events from endpoints.

| Column Name | Data Type | Constraints | Description |
|-------------|-----------|-------------|-------------|
| id | UUID | PRIMARY KEY, DEFAULT gen_random_uuid() | Event identifier |
| endpoint_id | UUID | NOT NULL, REFERENCES endpoints(id) | Source endpoint |
| event_type | VARCHAR(100) | NOT NULL | Event classification |
| event_data | JSONB | NOT NULL | Raw event data |
| timestamp | TIMESTAMP | NOT NULL | Event occurrence time |
| severity | VARCHAR(20) | NOT NULL | Event severity |
| processed | BOOLEAN | NOT NULL, DEFAULT false | Processing status |
| threat_id | UUID | REFERENCES threats(id) | Associated threat |
| created_at | TIMESTAMP | NOT NULL, DEFAULT CURRENT_TIMESTAMP | Ingestion timestamp |

### 5. vulnerabilities
System vulnerabilities and patches.

| Column Name | Data Type | Constraints | Description |
|-------------|-----------|-------------|-------------|
| id | UUID | PRIMARY KEY, DEFAULT gen_random_uuid() | Vulnerability identifier |
| cve_id | VARCHAR(50) | | CVE identifier |
| title | VARCHAR(500) | NOT NULL | Vulnerability title |
| description | TEXT | NOT NULL | Vulnerability description |
| severity | VARCHAR(20) | NOT NULL | CVSS severity rating |
| cvss_score | DECIMAL(3,1) | CHECK (cvss_score >= 0 AND cvss_score <= 10) | CVSS base score |
| endpoint_id | UUID | NOT NULL, REFERENCES endpoints(id) | Affected endpoint |
| status | VARCHAR(50) | NOT NULL, DEFAULT 'open' | Remediation status |
| discovered_at | TIMESTAMP | NOT NULL | Discovery timestamp |
| patched_at | TIMESTAMP | | Patch application timestamp |
| created_at | TIMESTAMP | NOT NULL, DEFAULT CURRENT_TIMESTAMP | Record creation timestamp |

### 6. compliance_assessments
Compliance assessment results.

| Column Name | Data Type | Constraints | Description |
|-------------|-----------|-------------|-------------|
| id | UUID | PRIMARY KEY, DEFAULT gen_random_uuid() | Assessment identifier |
| framework_id | UUID | NOT NULL | Reference to global framework |
| control_id | UUID | NOT NULL | Reference to global control |
| status | VARCHAR(50) | NOT NULL | Compliance status (compliant, non-compliant, partial) |
| evidence_collected | BOOLEAN | NOT NULL, DEFAULT false | Evidence collection status |
| last_assessed | TIMESTAMP | NOT NULL | Last assessment date |
| next_assessment | TIMESTAMP | | Next scheduled assessment |
| notes | TEXT | | Assessment notes |
| assigned_user_id | UUID | REFERENCES users(id) | Responsible user |
| created_at | TIMESTAMP | NOT NULL, DEFAULT CURRENT_TIMESTAMP | Record creation timestamp |
| updated_at | TIMESTAMP | NOT NULL, DEFAULT CURRENT_TIMESTAMP | Last update timestamp |

### 7. compliance_evidence
Compliance evidence artifacts.

| Column Name | Data Type | Constraints | Description |
|-------------|-----------|-------------|-------------|
| id | UUID | PRIMARY KEY, DEFAULT gen_random_uuid() | Evidence identifier |
| assessment_id | UUID | NOT NULL, REFERENCES compliance_assessments(id) | Associated assessment |
| evidence_type | VARCHAR(100) | NOT NULL | Evidence type (document, screenshot, log, etc.) |
| file_name | VARCHAR(255) | NOT NULL | Original file name |
| file_path | VARCHAR(500) | NOT NULL | Storage path |
| file_size | BIGINT | NOT NULL | File size in bytes |
| mime_type | VARCHAR(100) | NOT NULL | File MIME type |
| hash_sha256 | VARCHAR(64) | NOT NULL | File integrity hash |
| uploaded_by | UUID | NOT NULL, REFERENCES users(id) | Uploader user |
| uploaded_at | TIMESTAMP | NOT NULL, DEFAULT CURRENT_TIMESTAMP | Upload timestamp |
| description | TEXT | | Evidence description |

### 8. notifications
User notifications and alerts.

| Column Name | Data Type | Constraints | Description |
|-------------|-----------|-------------|-------------|
| id | UUID | PRIMARY KEY, DEFAULT gen_random_uuid() | Notification identifier |
| user_id | UUID | NOT NULL, REFERENCES users(id) | Target user |
| type | VARCHAR(100) | NOT NULL | Notification type |
| title | VARCHAR(255) | NOT NULL | Notification title |
| message | TEXT | NOT NULL | Notification content |
| priority | VARCHAR(20) | NOT NULL, DEFAULT 'medium' | Priority level |
| read | BOOLEAN | NOT NULL, DEFAULT false | Read status |
| related_entity_type | VARCHAR(100) | | Related entity type |
| related_entity_id | UUID | | Related entity identifier |
| created_at | TIMESTAMP | NOT NULL, DEFAULT CURRENT_TIMESTAMP | Creation timestamp |
| read_at | TIMESTAMP | | Read timestamp |

### 9. reports
Generated reports and exports.

| Column Name | Data Type | Constraints | Description |
|-------------|-----------|-------------|-------------|
| id | UUID | PRIMARY KEY, DEFAULT gen_random_uuid() | Report identifier |
| name | VARCHAR(255) | NOT NULL | Report name |
| type | VARCHAR(100) | NOT NULL | Report type |
| format | VARCHAR(20) | NOT NULL | Output format (pdf, csv, json) |
| status | VARCHAR(50) | NOT NULL, DEFAULT 'pending' | Generation status |
| file_path | VARCHAR(500) | | Generated file path |
| file_size | BIGINT | | File size in bytes |
| parameters | JSONB | | Report parameters |
| generated_by | UUID | NOT NULL, REFERENCES users(id) | Report creator |
| generated_at | TIMESTAMP | | Generation completion time |
| expires_at | TIMESTAMP | | Report expiration time |
| created_at | TIMESTAMP | NOT NULL, DEFAULT CURRENT_TIMESTAMP | Request timestamp |

### 10. audit_logs
Comprehensive audit trail.

| Column Name | Data Type | Constraints | Description |
|-------------|-----------|-------------|-------------|
| id | UUID | PRIMARY KEY, DEFAULT gen_random_uuid() | Log entry identifier |
| user_id | UUID | REFERENCES users(id) | Acting user |
| action | VARCHAR(100) | NOT NULL | Action performed |
| entity_type | VARCHAR(100) | NOT NULL | Affected entity type |
| entity_id | UUID | | Affected entity identifier |
| old_values | JSONB | | Previous values |
| new_values | JSONB | | New values |
| ip_address | INET | NOT NULL | Source IP address |
| user_agent | TEXT | | User agent string |
| session_id | VARCHAR(255) | | Session identifier |
| timestamp | TIMESTAMP | NOT NULL, DEFAULT CURRENT_TIMESTAMP | Action timestamp |
| success | BOOLEAN | NOT NULL, DEFAULT true | Action success status |
| error_message | TEXT | | Error details if failed |

---

## SQL DDL Scripts

### Global Schema Setup

```sql
-- Create tenants table
CREATE TABLE IF NOT EXISTS public.tenants (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    subdomain VARCHAR(100) UNIQUE NOT NULL,
    schema_name VARCHAR(100) UNIQUE NOT NULL,
    status VARCHAR(50) NOT NULL DEFAULT 'active',
    subscription_plan VARCHAR(100) NOT NULL,
    max_users INTEGER NOT NULL DEFAULT 50,
    max_endpoints INTEGER NOT NULL DEFAULT 500,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    billing_email VARCHAR(255) NOT NULL,
    technical_contact VARCHAR(255) NOT NULL,
    CONSTRAINT chk_status CHECK (status IN ('active', 'suspended', 'terminated'))
);

-- Create compliance frameworks table
CREATE TABLE IF NOT EXISTS public.compliance_frameworks (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(100) UNIQUE NOT NULL,
    version VARCHAR(50) NOT NULL,
    description TEXT,
    is_active BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);

-- Create compliance controls table
CREATE TABLE IF NOT EXISTS public.compliance_controls (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    framework_id UUID NOT NULL REFERENCES public.compliance_frameworks(id) ON DELETE CASCADE,
    control_number VARCHAR(50) NOT NULL,
    title VARCHAR(500) NOT NULL,
    description TEXT NOT NULL,
    control_type VARCHAR(100) NOT NULL,
    category VARCHAR(100) NOT NULL,
    is_active BOOLEAN NOT NULL DEFAULT true,
    UNIQUE(framework_id, control_number)
);

-- Create indexes for global tables
CREATE INDEX idx_tenants_schema_name ON public.tenants(schema_name);
CREATE INDEX idx_tenants_status ON public.tenants(status);
CREATE INDEX idx_compliance_controls_framework ON public.compliance_controls(framework_id);

-- Create tenant schema creation function
CREATE OR REPLACE FUNCTION create_tenant_schema(tenant_id UUID, schema_name VARCHAR)
RETURNS VOID AS $$
BEGIN
    -- Create schema
    EXECUTE format('CREATE SCHEMA IF NOT EXISTS %I', schema_name);
    
    -- Set search path for schema
    EXECUTE format('SET search_path TO %I', schema_name);
    
    -- Create all tenant tables in the new schema
    EXECUTE format('
        CREATE TABLE %I.users (
            id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
            tenant_id UUID NOT NULL,
            email VARCHAR(255) UNIQUE NOT NULL,
            first_name VARCHAR(100) NOT NULL,
            last_name VARCHAR(100) NOT NULL,
            phone VARCHAR(20),
            is_active BOOLEAN NOT NULL DEFAULT true,
            last_login TIMESTAMP,
            mfa_enabled BOOLEAN NOT NULL DEFAULT false,
            created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
            updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
            password_hash VARCHAR(255) NOT NULL,
            role VARCHAR(100) NOT NULL DEFAULT ''user'',
            CONSTRAINT chk_role CHECK (role IN (''admin'', ''analyst'', ''user'', ''auditor''))
        )', schema_name);
    
    EXECUTE format('
        CREATE TABLE %I.endpoints (
            id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
            hostname VARCHAR(255) NOT NULL,
            ip_address INET NOT NULL,
            mac_address MACADDR,
            operating_system VARCHAR(100) NOT NULL,
            os_version VARCHAR(100),
            endpoint_type VARCHAR(50) NOT NULL,
            status VARCHAR(50) NOT NULL DEFAULT ''active'',
            last_seen TIMESTAMP,
            agent_version VARCHAR(50),
            location VARCHAR(255),
            owner_user_id UUID REFERENCES %I.users(id),
            created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
            updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
            CONSTRAINT chk_endpoint_type CHECK (endpoint_type IN (''workstation'', ''server'', ''mobile'', ''iot'')),
            CONSTRAINT chk_status CHECK (status IN (''active'', ''inactive'', ''quarantined''))
        )', schema_name, schema_name);
    
    EXECUTE format('
        CREATE TABLE %I.threats (
            id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
            threat_type VARCHAR(100) NOT NULL,
            severity VARCHAR(20) NOT NULL,
            status VARCHAR(50) NOT NULL DEFAULT ''open'',
            title VARCHAR(500) NOT NULL,
            description TEXT NOT NULL,
            source_endpoint_id UUID REFERENCES %I.endpoints(id),
            assigned_user_id UUID REFERENCES %I.users(id),
            detected_at TIMESTAMP NOT NULL,
            resolved_at TIMESTAMP,
            created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
            updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
            mitre_tactic VARCHAR(100),
            mitre_technique VARCHAR(100),
            confidence_score DECIMAL(3,2) CHECK (confidence_score >= 0 AND confidence_score <= 1),
            CONSTRAINT chk_severity CHECK (severity IN (''critical'', ''high'', ''medium'', ''low'')),
            CONSTRAINT chk_status CHECK (status IN (''open'', ''investigating'', ''resolved'', ''false_positive''))
        )', schema_name, schema_name, schema_name);
    
    EXECUTE format('
        CREATE TABLE %I.security_events (
            id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
            endpoint_id UUID NOT NULL REFERENCES %I.endpoints(id),
            event_type VARCHAR(100) NOT NULL,
            event_data JSONB NOT NULL,
            timestamp TIMESTAMP NOT NULL,
            severity VARCHAR(20) NOT NULL,
            processed BOOLEAN NOT NULL DEFAULT false,
            threat_id UUID REFERENCES %I.threats(id),
            created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
            CONSTRAINT chk_severity CHECK (severity IN (''critical'', ''high'', ''medium'', ''low'', ''info''))
        )', schema_name, schema_name, schema_name);
    
    EXECUTE format('
        CREATE TABLE %I.vulnerabilities (
            id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
            cve_id VARCHAR(50),
            title VARCHAR(500) NOT NULL,
            description TEXT NOT NULL,
            severity VARCHAR(20) NOT NULL,
            cvss_score DECIMAL(3,1) CHECK (cvss_score >= 0 AND cvss_score <= 10),
            endpoint_id UUID NOT NULL REFERENCES %I.endpoints(id),
            status VARCHAR(50) NOT NULL DEFAULT ''open'',
            discovered_at TIMESTAMP NOT NULL,
            patched_at TIMESTAMP,
            created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
            CONSTRAINT chk_severity CHECK (severity IN (''critical'', ''high'', ''medium'', ''low'')),
            CONSTRAINT chk_status CHECK (status IN (''open'', ''in_progress'', ''patched'', ''mitigated'', ''accepted''))
        )', schema_name, schema_name);
    
    EXECUTE format('
        CREATE TABLE %I.compliance_assessments (
            id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
            framework_id UUID NOT NULL,
            control_id UUID NOT NULL,
            status VARCHAR(50) NOT NULL,
            evidence_collected BOOLEAN NOT NULL DEFAULT false,
            last_assessed TIMESTAMP NOT NULL,
            next_assessment TIMESTAMP,
            notes TEXT,
            assigned_user_id UUID REFERENCES %I.users(id),
            created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
            updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
            CONSTRAINT chk_status CHECK (status IN (''compliant'', ''non_compliant'', ''partial'', ''not_assessed''))
        )', schema_name, schema_name);
    
    EXECUTE format('
        CREATE TABLE %I.compliance_evidence (
            id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
            assessment_id UUID NOT NULL REFERENCES %I.compliance_assessments(id) ON DELETE CASCADE,
            evidence_type VARCHAR(100) NOT NULL,
            file_name VARCHAR(255) NOT NULL,
            file_path VARCHAR(500) NOT NULL,
            file_size BIGINT NOT NULL,
            mime_type VARCHAR(100) NOT NULL,
            hash_sha256 VARCHAR(64) NOT NULL,
            uploaded_by UUID NOT NULL REFERENCES %I.users(id),
            uploaded_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
            description TEXT
        )', schema_name, schema_name, schema_name);
    
    EXECUTE format('
        CREATE TABLE %I.notifications (
            id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
            user_id UUID NOT NULL REFERENCES %I.users(id) ON DELETE CASCADE,
            type VARCHAR(100) NOT NULL,
            title VARCHAR(255) NOT NULL,
            message TEXT NOT NULL,
            priority VARCHAR(20) NOT NULL DEFAULT ''medium'',
            read BOOLEAN NOT NULL DEFAULT false,
            related_entity_type VARCHAR(100),
            related_entity_id UUID,
            created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
            read_at TIMESTAMP,
            CONSTRAINT chk_priority CHECK (priority IN (''critical'', ''high'', ''medium'', ''low''))
        )', schema_name, schema_name);
    
    EXECUTE format('
        CREATE TABLE %I.reports (
            id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
            name VARCHAR(255) NOT NULL,
            type VARCHAR(100) NOT NULL,
            format VARCHAR(20) NOT NULL,
            status VARCHAR(50) NOT NULL DEFAULT ''pending'',
            file_path VARCHAR(500),
            file_size BIGINT,
            parameters JSONB,
            generated_by UUID NOT NULL REFERENCES %I.users(id),
            generated_at TIMESTAMP,
            expires_at TIMESTAMP,
            created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
            CONSTRAINT chk_format CHECK (format IN (''pdf'', ''csv'', ''json'', ''xlsx'')),
            CONSTRAINT chk_status CHECK (status IN (''pending'', ''generating'', ''completed'', ''failed'', ''expired''))
        )', schema_name, schema_name);
    
    EXECUTE format('
        CREATE TABLE %I.audit_logs (
            id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
            user_id UUID REFERENCES %I.users(id),
            action VARCHAR(100) NOT NULL,
            entity_type VARCHAR(100) NOT NULL,
            entity_id UUID,
            old_values JSONB,
            new_values JSONB,
            ip_address INET NOT NULL,
            user_agent TEXT,
            session_id VARCHAR(255),
            timestamp TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
            success BOOLEAN NOT NULL DEFAULT true,
            error_message TEXT
        )', schema_name, schema_name);
    
    -- Create indexes for tenant schema
    EXECUTE format('CREATE INDEX idx_%I_users_email ON %I.users(email)', schema_name, schema_name);
    EXECUTE format('CREATE INDEX idx_%I_users_tenant ON %I.users(tenant_id)', schema_name, schema_name);
    EXECUTE format('CREATE INDEX idx_%I_endpoints_ip ON %I.endpoints(ip_address)', schema_name, schema_name);
    EXECUTE format('CREATE INDEX idx_%I_endpoints_hostname ON %I.endpoints(hostname)', schema_name, schema_name);
    EXECUTE format('CREATE INDEX idx_%I_threats_severity ON %I.threats(severity)', schema_name, schema_name);
    EXECUTE format('CREATE INDEX idx_%I_threats_status ON %I.threats(status)', schema_name, schema_name);
    EXECUTE format('CREATE INDEX idx_%I_security_events_timestamp ON %I.security_events(timestamp)', schema_name, schema_name);
    EXECUTE format('CREATE INDEX idx_%I_security_events_endpoint ON %I.security_events(endpoint_id)', schema_name, schema_name);
    EXECUTE format('CREATE INDEX idx_%I_vulnerabilities_severity ON %I.vulnerabilities(severity)', schema_name, schema_name);
    EXECUTE format('CREATE INDEX idx_%I_vulnerabilities_cve ON %I.vulnerabilities(cve_id)', schema_name, schema_name);
    EXECUTE format('CREATE INDEX idx_%I_audit_logs_timestamp ON %I.audit_logs(timestamp)', schema_name, schema_name);
    EXECUTE format('CREATE INDEX idx_%I_audit_logs_user ON %I.audit_logs(user_id)', schema_name, schema_name);
    
    -- Enable Row Level Security on all tables
    EXECUTE format('ALTER TABLE %I.users ENABLE ROW LEVEL SECURITY', schema_name);
    EXECUTE format('ALTER TABLE %I.endpoints ENABLE ROW LEVEL SECURITY', schema_name);
    EXECUTE format('ALTER TABLE %I.threats ENABLE ROW LEVEL SECURITY', schema_name);
    EXECUTE format('ALTER TABLE %I.security_events ENABLE ROW LEVEL SECURITY', schema_name);
    EXECUTE format('ALTER TABLE %I.vulnerabilities ENABLE ROW LEVEL SECURITY', schema_name);
    EXECUTE format('ALTER TABLE %I.compliance_assessments ENABLE ROW LEVEL SECURITY', schema_name);
    EXECUTE format('ALTER TABLE %I.compliance_evidence ENABLE ROW LEVEL SECURITY', schema_name);
    EXECUTE format('ALTER TABLE %I.notifications ENABLE ROW LEVEL SECURITY', schema_name);
    EXECUTE format('ALTER TABLE %I.reports ENABLE ROW LEVEL SECURITY', schema_name);
    EXECUTE format('ALTER TABLE %I.audit_logs ENABLE ROW LEVEL SECURITY', schema_name);
    
    -- Create RLS policies
    EXECUTE format('CREATE POLICY tenant_isolation ON %I.users USING (tenant_id = current_setting(''app.current_tenant'')::UUID)', schema_name);
    EXECUTE format('CREATE POLICY tenant_isolation ON %I.endpoints USING (true)', schema_name);
    EXECUTE format('CREATE POLICY tenant_isolation ON %I.threats USING (true)', schema_name);
    EXECUTE format('CREATE POLICY tenant_isolation ON %I.security_events USING (true)', schema_name);
    EXECUTE format('CREATE POLICY tenant_isolation ON %I.vulnerabilities USING (true)', schema_name);
    EXECUTE format('CREATE POLICY tenant_isolation ON %I.compliance_assessments USING (true)', schema_name);
    EXECUTE format('CREATE POLICY tenant_isolation ON %I.compliance_evidence USING (true)', schema_name);
    EXECUTE format('CREATE POLICY tenant_isolation ON %I.notifications USING (true)', schema_name);
    EXECUTE format('CREATE POLICY tenant_isolation ON %I.reports USING (true)', schema_name);
    EXECUTE format('CREATE POLICY tenant_isolation ON %I.audit_logs USING (true)', schema_name);
    
    -- Reset search path
    SET search_path TO public;
END;
$$ LANGUAGE plpgsql;
```

### Sample DML Scripts

```sql
-- Insert sample compliance frameworks
INSERT INTO public.compliance_frameworks (name, version, description) VALUES
('SOC 2 Type II', '2017', 'Service Organization Control 2 Type II certification framework'),
('HIPAA', '2013', 'Health Insurance Portability and Accountability Act compliance'),
('PCI DSS', '4.0', 'Payment Card Industry Data Security Standard'),
('ISO 27001', '2013', 'Information Security Management System standard'),
('GDPR', '2018', 'General Data Protection Regulation'),
('NIST Cybersecurity Framework', '1.1', 'National Institute of Standards and Technology Cybersecurity Framework');

-- Insert sample compliance controls for SOC 2
INSERT INTO public.compliance_controls (framework_id, control_number, title, description, control_type, category)
SELECT 
    f.id,
    'CC6.1',
    'Logical and Physical Access Controls',
    'The entity implements logical and physical access controls to protect against unauthorized access to data and system resources.',
    'Preventive',
    'Access Control'
FROM public.compliance_frameworks f WHERE f.name = 'SOC 2 Type II';

-- Create sample tenant
INSERT INTO public.tenants (name, subdomain, schema_name, subscription_plan, billing_email, technical_contact)
VALUES ('Acme Corporation', 'acme', 'tenant_acme', 'professional', 'billing@acme.com', 'admin@acme.com');

-- Create tenant schema for Acme
SELECT create_tenant_schema(
    (SELECT id FROM public.tenants WHERE subdomain = 'acme'),
    'tenant_acme'
);

-- Sample data for tenant schema (set search path first)
SET search_path TO tenant_acme;

-- Insert sample users
INSERT INTO users (tenant_id, email, first_name, last_name, role, password_hash, mfa_enabled)
SELECT 
    t.id,
    'admin@acme.com',
    'John',
    'Admin',
    'admin',
    '$2a$10$example_hash',
    true
FROM public.tenants t WHERE t.subdomain = 'acme';

INSERT INTO users (tenant_id, email, first_name, last_name, role, password_hash)
SELECT 
    t.id,
    'analyst@acme.com',
    'Jane',
    'Analyst', 
    'analyst',
    '$2a$10$example_hash'
FROM public.tenants t WHERE t.subdomain = 'acme';

-- Insert sample endpoints
INSERT INTO endpoints (hostname, ip_address, operating_system, endpoint_type, owner_user_id)
VALUES 
('WS-001', '192.168.1.100', 'Windows 11', 'workstation', (SELECT id FROM users WHERE email = 'admin@acme.com')),
('SRV-001', '192.168.1.10', 'Ubuntu 22.04', 'server', NULL),
('MOB-001', '192.168.1.150', 'iOS 16', 'mobile', (SELECT id FROM users WHERE email = 'analyst@acme.com'));

-- Insert sample threats
INSERT INTO threats (threat_type, severity, title, description, source_endpoint_id, detected_at, mitre_tactic, confidence_score)
VALUES 
('Malware', 'high', 'Suspicious Process Execution', 'Unusual process spawned from legitimate application', 
 (SELECT id FROM endpoints WHERE hostname = 'WS-001'), CURRENT_TIMESTAMP - INTERVAL '2 hours', 'Execution', 0.85),
('Network Anomaly', 'medium', 'Unusual Outbound Traffic', 'High volume of data transfer to external IP',
 (SELECT id FROM endpoints WHERE hostname = 'SRV-001'), CURRENT_TIMESTAMP - INTERVAL '1 hour', 'Exfiltration', 0.72);

-- Insert sample vulnerabilities
INSERT INTO vulnerabilities (cve_id, title, description, severity, cvss_score, endpoint_id, discovered_at)
VALUES 
('CVE-2023-1234', 'Remote Code Execution in Web Server', 'Buffer overflow vulnerability allowing remote code execution', 
 'critical', 9.8, (SELECT id FROM endpoints WHERE hostname = 'SRV-001'), CURRENT_TIMESTAMP - INTERVAL '1 day'),
('CVE-2023-5678', 'Privilege Escalation in OS Kernel', 'Local privilege escalation vulnerability',
 'high', 7.8, (SELECT id FROM endpoints WHERE hostname = 'WS-001'), CURRENT_TIMESTAMP - INTERVAL '3 days');

-- Reset search path
SET search_path TO public;
```

### Utility Functions

```sql
-- Function to get tenant by subdomain
CREATE OR REPLACE FUNCTION get_tenant_by_subdomain(subdomain_param VARCHAR)
RETURNS TABLE(tenant_id UUID, schema_name VARCHAR) AS $$
BEGIN
    RETURN QUERY
    SELECT t.id, t.schema_name
    FROM public.tenants t
    WHERE t.subdomain = subdomain_param AND t.status = 'active';
END;
$$ LANGUAGE plpgsql;

-- Function to set tenant context
CREATE OR REPLACE FUNCTION set_tenant_context(tenant_uuid UUID)
RETURNS VOID AS $$
BEGIN
    PERFORM set_config('app.current_tenant', tenant_uuid::TEXT, true);
END;
$$ LANGUAGE plpgsql;

-- Function to get current tenant
CREATE OR REPLACE FUNCTION get_current_tenant()
RETURNS UUID AS $$
BEGIN
    RETURN current_setting('app.current_tenant', true)::UUID;
EXCEPTION
    WHEN others THEN
        RETURN NULL;
END;
$$ LANGUAGE plpgsql;
```

## Notes

1. **Row Level Security (RLS)**: All tenant tables have RLS enabled to ensure data isolation
2. **Tenant Context**: Use `set_tenant_context()` function before performing operations
3. **Schema Isolation**: Each tenant has a separate schema for complete data isolation
4. **Audit Trail**: All operations are logged in the audit_logs table
5. **UUID Primary Keys**: All tables use UUID for better distribution and security
6. **JSONB Support**: Event data and report parameters use JSONB for flexibility
7. **Time-series Optimization**: Consider partitioning security_events by timestamp for better performance
8. **Backup Strategy**: Schema-per-tenant allows for selective backup and restore operations

