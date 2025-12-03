/******************************************************************************************
 * @Description: * utility Factory for test data creation
 * @Author: Arya Shah (Deloitte USI)
 * @Date : 10-Nov-2025
 * @Test Class: 
 * ******************************************************************************************
 * ModificationLog      Developer           CodeReview          Date            Description
 * 1.0                  Arya Shah                               10-Nov-2025     Initial Version
 * ******************************************************************************************/
@isTest
public with sharing class AF_TestDataFactory {
    //Creates and inserts an Account.
    public static Account createAccount(Map<String, Object> accInputMap){
        Account acc = new Account(Name = 'Test Account');
        if (accInputMap != null){
            for (String key : accInputMap.keySet()){
                acc.put(key, accInputMap.get(key));
            }
        }
        insert acc;
        return acc;
    }

    //Creates and inserts a Contact for a given Account
    public static Contact createContact(Map<String, Object> conInputMap, Id accountId){
        Contact con = new Contact(LastName = 'Test Contact', AccountId = accountId);
        if (conInputMap != null){
            for(String key : conInputMap.keySet()){
                con.put(key, conInputMap.get(key));
            }
        }
        insert con;
        return con;
    }

    //Creates and inserts a Task.
    public static Task createTask(Map<String, Object> taskInputMap){
        Task t = new Task(
            Subject = 'Default Task',
            Status = 'Not Started',
            Priority = 'Normal',
            Description = 'This is to createtask',
            ActivityDate = Date.today()
        );
        if(taskInputMap != null){
            for (String key : taskInputMap.keySet()){
                t.put(key, taskInputMap.get(key));
            }
        }
        insert t; 
        return t;
    }

    // Creates and inserts an Opportunity.
    public static Opportunity createOpportunity(Map<String, Object> oppInputMap){
        Opportunity opp = new Opportunity(
            Name = 'Default Opp',
            StageName = 'Pre-Discovery',
            CloseDate = Date.today().addDays(90),
            LeadSource = 'Other',
            Legacy_Id__c = 'Agentforce'
        );
        if(oppInputMap != null){
            for (String key : oppInputMap.keySet()){
                opp.put(key, oppInputMap.get(key));
            }
        }
        insert opp;
        return opp;
    }

    /**Create Meeting Note Action custom object record **/
    public static AF_Meeting_Note_Action__c createMeetingNoteAction(String meetingNoteId, String objName, String status, String actionName, String recordId){
        AF_Meeting_Note_Action__c mna = new AF_Meeting_Note_Action__c(
            AF_Meeting_Note__c = meetingNoteId,
            AF_Object_Name__c = objName,
            AF_Status__c = status,
            AF_Action_Name__c = actionName,
            AF_Record_ID__c = recordId
        );
        insert mna;
        return mna;
    }

    /**Create User object record **/
    public static User createUser(Map<String, Object> userInputMap) {
        Profile p = [SELECT Id FROM Profile WHERE Name = 'Standard User' LIMIT 1];
        User u = new User(
            Alias = 'tusr',
            Email = 'user'+System.currentTimeMillis()+'@testorg.com',
            EmailEncodingKey = 'UTF-8',
            LastName = 'TestUser',
            LanguageLocaleKey = 'en_US',
            LocaleSidKey = 'en_US',
            ProfileId = p.Id,
            TimeZoneSidKey = 'America/New_York',
            Username = 'user'+System.currentTimeMillis()+'@testorg.com'
        );
        if (userInputMap != null) {
            for (String key : userInputMap.keySet()) {
                u.put(key, userInputMap.get(key));
            }
        }
        insert u;
        return u;
    }

    // Create Meeting Note Action Data custom object record **/
    public static AF_Meeting_Note_Action_Data__c createMeetingNoteActionData(Id actionId, String fieldName, String val) {
        AF_Meeting_Note_Action_Data__c data = new AF_Meeting_Note_Action_Data__c(
            AF_Meeting_Note_Action__c = actionId,
            AF_Field_Name__c = fieldName,
            AF_Field_Value__c = val
        );
        insert data;
        return data;
    }
}

