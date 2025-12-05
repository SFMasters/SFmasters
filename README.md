
Step 1: Create Named Credential
Setup → Named Credentials & External Credentials → Named Credentials → New

Settings:

Label: OrgSelfCallout
Name: OrgSelfCallout
URL: https://sdev2512.bacm1domadv.sandbox.my-salesforce.com
Identity Type: Per User
Authentication Protocol: Password Authentication
Username: Your admin Salesforce username
Password: Your password + security token
Save
Step 2: Update Apex Code
In getRecordTypePicklistValues method, replace authentication section with:

// Use Named Credential endpoint - handles authentication automatically
String endpoint = 'callout:OrgSelfCallout/services/data/v58.0/ui-api/object-info/' + objectName + 
                '/picklist-values/' + recordTypeId + '/' + fieldName;

// Create HTTP request - no manual authentication needed
HttpRequest req = new HttpRequest();
req.setEndpoint(endpoint);
req.setHeader('Content-Type', 'application/json');
req.setMethod('GET');
req.setTimeout(10000);

System.debug('Named Credential Endpoint: ' + endpoint);
