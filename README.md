/** Create Lead with overrides **/
    public static Lead createLead(Map<String, Object> overrides) {
        Lead l = new Lead(
            FirstName = 'Test',
            LastName = 'Lead',
            Company = 'Test Company'
        );
        if (overrides != null) {
            for (String key : overrides.keySet()) {
                l.put(key, overrides.get(key));
            }
        }
        insert l;
        return l;
    }

    /** Create MeetingNoteActionCreator Request - reduces test code repetition **/
    public static AF_MeetingNoteActionCreator.Request createMeetingNoteActionRequest(
        String objectApiName, String taskId, String relatedRecordId, String actionName, String fieldData) {
        AF_MeetingNoteActionCreator.Request req = new AF_MeetingNoteActionCreator.Request();
        req.objectApiName = objectApiName;
        req.taskId = taskId;
        req.relatedRecordId = relatedRecordId;
        req.actionName = actionName;
        req.fieldApiNameToValueJSON = fieldData;
        return req;
    }
	
	
	
	
	
	// JIRA-265: Ultimate optimized test - all filtering scenarios with user context
    @isTest
    static void testJIRA265_OpportunityFiltering_AllScenarios() {
        User testUser = AF_TestDataFactory.createUser(new Map<String, Object>{'LastName' => 'JIRA265 User'});
        
        System.runAs(testUser) {
            Account acc = AF_TestDataFactory.createAccount(new Map<String, Object>{'Name' => 'JIRA265 Test Account'});
            Contact testContact = AF_TestDataFactory.createContact(new Map<String, Object>{'FirstName' => 'JIRA265'}, acc.Id);
            Lead testLead = AF_TestDataFactory.createLead(new Map<String, Object>{'FirstName' => 'JIRA265', 'LastName' => 'Lead'});
            
            // Create different task scenarios using factory
            Task contactOnlyTask = AF_TestDataFactory.createTask(new Map<String, Object>{
                'Subject' => 'Contact-only Meeting', 'WhoId' => testContact.Id, 'WhatId' => null
            });
            Task leadOnlyTask = AF_TestDataFactory.createTask(new Map<String, Object>{
                'Subject' => 'Lead-only Meeting', 'WhoId' => testLead.Id, 'WhatId' => null
            });
            Task contactAccountTask = AF_TestDataFactory.createTask(new Map<String, Object>{
                'Subject' => 'Contact+Account Meeting', 'WhoId' => testContact.Id, 'WhatId' => acc.Id
            });

            String oppFieldJSON = JSON.serialize(new List<Object>{new Map<String, Object>{'Name' => 'Test Opportunity'}});
            String taskFieldJSON = JSON.serialize(new List<Object>{new Map<String, Object>{'Subject' => 'Follow-up Task'}});
            
            // Use factory methods to create requests - eliminates repetition
            List<AF_MeetingNoteActionCreator.Request> requests = new List<AF_MeetingNoteActionCreator.Request>{
                AF_TestDataFactory.createMeetingNoteActionRequest('Opportunity', String.valueOf(contactOnlyTask.Id), String.valueOf(acc.Id), 'Contact-only Opportunity', oppFieldJSON),
                AF_TestDataFactory.createMeetingNoteActionRequest('Opportunity', String.valueOf(leadOnlyTask.Id), String.valueOf(acc.Id), 'Lead-only Opportunity', oppFieldJSON),
                AF_TestDataFactory.createMeetingNoteActionRequest('Opportunity', String.valueOf(contactAccountTask.Id), String.valueOf(acc.Id), 'Allowed Opportunity', oppFieldJSON),
                AF_TestDataFactory.createMeetingNoteActionRequest('Task', String.valueOf(contactOnlyTask.Id), String.valueOf(acc.Id), 'Contact-only Task', taskFieldJSON)
            };

            Test.startTest();
            List<AF_MeetingNoteActionCreator.Response> responses = AF_MeetingNoteActionCreator.createAction(requests);
            Test.stopTest();

            // Optimized assertions - descriptive and debuggable
            System.assertEquals(4, responses.size(), 'Should have 4 responses');
            
            // Scenario 1: Contact-only + Opportunity → Blocked
            System.assert(responses[0].message.contains('Skipped: Opportunity creation not allowed'), 
                         'Contact-only Opportunity should be blocked');
            System.assertEquals(null, responses[0].tempActionId, 'Contact-only Opportunity should have null tempActionId');
            
            // Scenario 2: Lead-only + Opportunity → Blocked  
            System.assert(responses[1].message.contains('Skipped: Opportunity creation not allowed'), 
                         'Lead-only Opportunity should be blocked');
            System.assertEquals(null, responses[1].tempActionId, 'Lead-only Opportunity should have null tempActionId');
            
            // Scenario 3: Contact+Account + Opportunity → Allowed
            System.assertEquals('Successfully created the temp action!', responses[2].message, 
                               'Contact+Account Opportunity should succeed');
            System.assertNotEquals(null, responses[2].tempActionId, 'Contact+Account Opportunity should have tempActionId');
            
            // Scenario 4: Contact-only + Task → Allowed
            System.assertEquals('Successfully created the temp action!', responses[3].message, 
                               'Contact-only Task should succeed');
            System.assertNotEquals(null, responses[3].tempActionId, 'Contact-only Task should have tempActionId');
        }
    }
