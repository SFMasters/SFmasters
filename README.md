Remote Site Name: OrgAPIDomain
Remote Site URL: https://sdev2512.bacm1domadv.sandbox.my-salesforce.com
Active:  Checked
Step 2: Try Different Authentication
Replace authentication in getRecordTypePicklistValues method:

Change:

req.setHeader('Authorization', 'Bearer ' + UserInfo.getSessionId());
To:

String sessionId = UserInfo.getSessionId();
req.setHeader('Authorization', 'OAuth ' + sessionId);
