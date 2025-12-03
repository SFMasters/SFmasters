// US-58: PersonLifeEvent filtering logic - all scenarios
@isTest
static void testUS58_PersonLifeEvent_AllFilteringScenarios() {
    User testUser = AF_TestDataFactory.createUser(new Map<String, Object>{'LastName' => 'US58 User'});
    
    System.runAs(testUser) {
        Account acc = AF_TestDataFactory.createAccount(new Map<String, Object>{'Name' => 'US58 Test Account'});
        Contact testContact = AF_TestDataFactory.createContact(new Map<String, Object>{'FirstName' => 'US58'}, acc.Id);
        Lead testLead = AF_TestDataFactory.createLead(new Map<String, Object>{'FirstName' => 'US58', 'LastName' => 'Lead'});
        
        // Create test tasks for different scenarios
        Task leadTask = AF_TestDataFactory.createTask(new Map<String, Object>{
            'Subject' => 'Lead Meeting', 'WhoId' => testLead.Id, 'WhatId' => null
        });
        Task contactTask = AF_TestDataFactory.createTask(new Map<String, Object>{
            'Subject' => 'Contact Meeting', 'WhoId' => testContact.Id, 'WhatId' => null
        });
        Task nullWhoIdTask = AF_TestDataFactory.createTask(new Map<String, Object>{
            'Subject' => 'Null WhoId Meeting', 'WhoId' => null, 'WhatId' => acc.Id
        });

        // Create existing PersonLifeEvent for duplicate testing
        PersonLifeEvent existingEvent = new PersonLifeEvent(
            PrimaryPersonId = testContact.Id,
            EventType = 'Birthday',
            EventDate = Date.today()
        );
        insert existingEvent;

        // Test data - PersonLifeEvent field JSON
        String duplicateEventJSON = JSON.serialize(new List<Object>{
            new Map<String, Object>{'EventType' => 'Birthday', 'EventDate' => String.valueOf(Date.today())}
        });
        String newEventJSON = JSON.serialize(new List<Object>{
            new Map<String, Object>{'EventType' => 'Anniversary', 'EventDate' => String.valueOf(Date.today())}
        });

        // Create test requests using helper method
        List<AF_MeetingNoteActionCreator.Request> requests = new List<AF_MeetingNoteActionCreator.Request>{
            createMeetingNoteActionRequest('PersonLifeEvent', String.valueOf(leadTask.Id), null, 'Lead PersonLifeEvent', newEventJSON),
            createMeetingNoteActionRequest('PersonLifeEvent', String.valueOf(nullWhoIdTask.Id), null, 'Null WhoId PersonLifeEvent', newEventJSON),
            createMeetingNoteActionRequest('PersonLifeEvent', String.valueOf(contactTask.Id), testContact.Id, 'Duplicate PersonLifeEvent', duplicateEventJSON),
            createMeetingNoteActionRequest('PersonLifeEvent', String.valueOf(contactTask.Id), testContact.Id, 'New PersonLifeEvent', newEventJSON)
        };

        Test.startTest();
        List<AF_MeetingNoteActionCreator.Response> responses = AF_MeetingNoteActionCreator.createAction(requests);
        Test.stopTest();

        // Verify responses
        System.assertEquals(4, responses.size(), 'Should have 4 responses');
        
        // Scenario 1: Lead WhoId → Blocked
        System.assert(responses[0].message.contains('Skipped: WhoId is a Lead'), 
                     'Lead PersonLifeEvent should be blocked');
        System.assertEquals(null, responses[0].tempActionId, 'Lead PersonLifeEvent should have null tempActionId');
        
        // Scenario 2: Null WhoId → Blocked
        System.assert(responses[1].message.contains('Skipped: WhoId is a Lead'), 
                     'Null WhoId PersonLifeEvent should be blocked');
        System.assertEquals(null, responses[1].tempActionId, 'Null WhoId PersonLifeEvent should have null tempActionId');
        
        // Scenario 3: Contact with duplicate PersonLifeEvent → Blocked
        System.assert(responses[2].message.contains('Skipped: PersonLifeEvent already exists'), 
                     'Duplicate PersonLifeEvent should be blocked');
        System.assertEquals(null, responses[2].tempActionId, 'Duplicate PersonLifeEvent should have null tempActionId');
        
        // Scenario 4: Contact with new PersonLifeEvent → Allowed
        System.assertEquals('Successfully created the temp action!', responses[3].message, 
                           'New PersonLifeEvent should succeed');
        System.assertNotEquals(null, responses[3].tempActionId, 'New PersonLifeEvent should have tempActionId');
    }
}
