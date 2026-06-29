## Malicious IP Protection Summary

Snowflake's Malicious IP Protection service continuously tracks and blocks inbound network access attempts originating from a curated intelligence list of known bad actors.

### 1. Threat Classifications
The service screens both IPv4 and IPv6 traffic across four distinct categories:

* **ANONYMOUS_VPN**: Requests routed through anonymous VPN services.
* **ANONYMOUS_PROXIES**: Requests masking through anonymous proxy servers.
* **MALICIOUS_BEHAVIOR**: Known malware or automated brute-force login vectors.
* **TOR_EXITS**: Traffic originating from Tor network exit nodes.

### 2. Auditing Blocked Traffic
Security administrators can track blocked connection events by querying account usage logs. Failed attempts display NO in the success column, while the specific reason and threat categories are logged as JSON payload metadata.
```sql
SELECT ip_address, login_details FROM SNOWFLAKE.ACCOUNT_USAGE.LOGIN_HISTORY WHERE NOT is_success 
  AND login_details IS NOT NULL ORDER BY event_timestamp DESC;
```
### 3. Managing Exemptions
Snowflake permits accounts to bypass restrictions only for classifications evaluated as low-risk. High-risk categories cannot be deactivated. Managing these exceptions requires the ACCOUNTADMIN role:

* Opt out of a category account-wide:
```sql
SELECT SYSTEM$OPT_OUT_MALICIOUS_IP_PROTECTION_BY_CATEGORY('MALICIOUS_BEHAVIOR');
```
* Opt out of multiple categories:
```sql
SELECT SYSTEM$OPT_OUT_MALICIOUS_IP_PROTECTION_BY_CATEGORY('ANONYMOUS_VPN,MALICIOUS_BEHAVIOR');
```
* Isolate an opt-out to a single user:
```sql
SELECT SYSTEM$OPT_OUT_MALICIOUS_IP_PROTECTION_BY_CATEGORY('ANONYMOUS_VPN', 'JSMITH');
```
* Reset and re-enable all default blocking:
```sql
SELECT SYSTEM$OPT_OUT_MALICIOUS_IP_PROTECTION_BY_CATEGORY('');
```
