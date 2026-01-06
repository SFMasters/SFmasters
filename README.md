   @isTest
    static void testUS58_PersonLifeEvent_UpdatedFilteringAllScenarios() {
        User testUser = AF_TestDataFactory.createUser(new Map<String, Object>{'LastName' => 'US58 User'});
        
        System.runAs(testUser) {
            // Standard Account for basic scenarios
            Account acc = AF_TestDataFactory.createAccount(new Map<String, Object>{'Name' => 'Standard Account'});
            Contact testContact = AF_TestDataFactory.createContact(new Map<String, Object>{'FirstName' => 'Test'}, acc.Id);
            Lead testLead = AF_TestDataFactory.createLead(new Map<String, Object>{'FirstName' => 'Test', 'LastName' => 'Lead'});
            
            // Advisory Individual Account for uncovered lines testing
            RecordType advisoryRT = [SELECT Id FROM RecordType WHERE SObjectType = 'Account' AND Name = 'Advisory Individual' LIMIT 1];
            Account advisoryAccount = AF_TestDataFactory.createAccount(new Map<String, Object>{
                'Name' => 'Advisory Individual Account', 'RecordTypeId' => advisoryRT.Id
            });
            Contact advisoryContact = AF_TestDataFactory.createContact(new Map<String, Object>{'FirstName' => 'Advisory'}, advisoryAccount.Id);
            
            // Core test tasks - reduced from 5 to 3 essential ones
            Task contactOnlyTask = AF_TestDataFactory.createTask(new Map<String, Object>{
                'Subject' => 'Contact-only Meeting', 'WhoId' => testContact.Id, 'WhatId' => null
            });
            Task leadOnlyTask = AF_TestDataFactory.createTask(new Map<String, Object>{
                'Subject' => 'Lead-only Meeting', 'WhoId' => testLead.Id, 'WhatId' => null
            });
            Task advisoryTask = AF_TestDataFactory.createTask(new Map<String, Object>{
                'Subject' => 'Advisory Meeting', 'WhoId' => advisoryContact.Id, 'WhatId' => advisoryAccount.Id
            });

            // Single existing PersonLifeEvent for duplicate testing
            // Note: Name field is auto-generated, so we only set writable fields
            PersonLifeEvent existingEvent = new PersonLifeEvent(
                PrimaryPersonId = advisoryContact.Id, EventType = 'Birthday', EventDate = Date.today()
            );
            insert existingEvent;

            // Optimized test data - only essential JSON strings
            String validEventJSON = JSON.serialize(new List<Object>{new Map<String, Object>{'Name' => 'Valid Event', 'EventType' => 'Anniversary', 'EventDate' => String.valueOf(Date.today())}});
            String blankEventTypeJSON = JSON.serialize(new List<Object>{new Map<String, Object>{'Name' => 'Blank Event', 'EventType' => '', 'EventDate' => String.valueOf(Date.today())}});
            String duplicateEventJSON = JSON.serialize(new List<Object>{new Map<String, Object>{'Name' => 'Duplicate Birthday', 'EventType' => 'Birthday', 'EventDate' => String.valueOf(Date.today())}});

            // Focused test requests - 6 key scenarios (reduced from 9)
            List<AF_MeetingNoteActionCreator.Request> requests = new List<AF_MeetingNoteActionCreator.Request>{
                // Core filtering scenarios (3)
                createMeetingNoteActionRequest('PersonLifeEvent', String.valueOf(contactOnlyTask.Id), testContact.Id, 'Contact-only', validEventJSON),
                createMeetingNoteActionRequest('PersonLifeEvent', String.valueOf(leadOnlyTask.Id), testLead.Id, 'Lead-only', validEventJSON),
                createMeetingNoteActionRequest('Task', String.valueOf(contactOnlyTask.Id), testContact.Id, 'Task Control', JSON.serialize(new List<Object>{new Map<String, Object>{'Subject' => 'Follow-up'}})),
                
                // Advisory Individual scenarios for uncovered lines (3)
                createMeetingNoteActionRequest('PersonLifeEvent', String.valueOf(advisoryTask.Id), advisoryContact.Id, 'Advisory Valid', validEventJSON),
                createMeetingNoteActionRequest('PersonLifeEvent', String.valueOf(advisoryTask.Id), advisoryContact.Id, 'Advisory Blank EventType', blankEventTypeJSON),
                createMeetingNoteActionRequest('PersonLifeEvent', String.valueOf(advisoryTask.Id), advisoryContact.Id, 'Advisory Duplicate', duplicateEventJSON)
            };

            Test.startTest();
            List<AF_MeetingNoteActionCreator.Response> responses = AF_MeetingNoteActionCreator.createAction(requests);
            Test.stopTest();

            // Streamlined assertions - essential validation only
            System.assertEquals(6, responses.size(), 'Should have 6 responses');
            
            // Core filtering validation
            System.assert(responses[0].message.contains('Skipped: PersonLifeEvent creation not allowed'), 'Contact-only should be blocked');
            System.assert(responses[1].message.contains('Skipped: WhoId is a Lead'), 'Lead should be blocked');
            System.assertEquals('Successfully created the temp action!', responses[2].message, 'Task should succeed');
            
            // Advisory Individual scenarios - CRITICAL for covering uncovered lines
            System.assertEquals('Successfully created the temp action!', responses[3].message, 'Advisory valid event should succeed');
            System.assert(responses[4].message.contains('Skipped: Event type is blank'), 'Blank EventType should trigger SKIP_EVENT_TYPE_BLANK');
            System.assert(responses[5].message.contains('Skipped: PersonLifeEvent already exists'), 'Duplicate should trigger SKIP_MILESTONE_EXIST');
        }
    }
