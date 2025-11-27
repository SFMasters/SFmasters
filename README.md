/**
 * @Description: * Creates Actual objects into database from temporary objects
 * @Author: Sujaanaa (Deloitte USI)
 * @Date : 24-Oct-2025
 * @Test Class: AF_MeetingNoteActionRunner_Test
 * ******************************************************************************************
 * ModificationLog      Developer           CodeReview          Date            Description
 * 1.0                  Sujaanaa                                                
 * 2.0                  Satis                                   20-Nov-2025     Added two methods to retrieve picklist values of Task
 * ******************************************************************************************
 */
public with sharing class AF_MeetingNoteActionRunner {

    public static final String OPPORTUNITY = 'Opportunity';
    public static final String OWNER_ID = 'OwnerId';
    public static final String STAGE_NAME = 'StageName';
    public static final String STAGE_PRE_DISCOVERY = 'Pre-Discovery';
    public static final String LEGACY_ID = 'Legacy_Id__c';
    public static final String LEGACY_ID_VAL = 'Agentforce';
    public static final String LEAD_SOURCE = 'LeadSource';
    public static final String LEAD_SOURCE_OTHER = 'Other';
    public static final String CLOSE_DATE = 'CloseDate';

    public static final String TASK = 'Task';
    public static final String RECORD_TYPE_ID = 'RecordTypeId';
    public static final String TASK_DEV_NAME = 'Advisory_Task';
    public static final String TYPE = 'Type';
    public static final String PRIORITY = 'Priority';
    public static final String TYPE_TODO = 'To Do';
    public static final String STATUS = 'Status';
    public static final String STATUS_NOT_STARTED = 'Not Started';
    public static final String WHAT_ID = 'WhatId';
    public static final String WHO_ID = 'WhoId';
    public static final String LINE_OF_BUSINESS = 'Line_Of_Business__c';
    public static final String LINE_OF_BUSINESS_VALUE = 'Advisory';
    public static final String START_DATE = 'OK_Start_Date__c';

    /**
     * Method name : runAction
     * @description : Method to Create dynamic objects based on action records
     * @return : Map<String, Object>
     * @param : String objectApiName : the record api name to be created, Map<String, Object> fieldValues :values of the fields, String taskId: Id of th...
     */
    @AuraEnabled
    public static Map<String, Object> runAction(String objectApiName, Map<String, Object> fieldValues, String taskId, String actionId) {
        Map<String, Object> result = new Map<String, Object>();
        try {
            SObjectType subjectType = Schema.getGlobalDescribe().get(objectApiName);
            SObject newObj = subjectType.newSObject();
            Map<String, SObjectField> fieldMap = subjectType.getDescribe().fields.getMap();

            populateSObjectFields(newObj, fieldMap, fieldValues);

            if (objectApiName == TASK) {
                setTaskDefaults(newObj, fieldMap, fieldValues, taskId);
            } else if (objectApiName == OPPORTUNITY) {
                setOpportunityDefaults(newObj, fieldMap, fieldValues);
            }

            insert newObj;
            result.put('objectLabel', subjectType.getDescribe().getLabel());
            result.put('recordId', newObj.Id);
            result.put('errorMessage', null);

            if (!String.isBlank(actionId)) {
                cleanupActionAndActionData(actionId, result, fieldValues);
            }
        } catch (Exception e) {
            result.put('objectLabel', null);
            result.put('recordId', null);
            result.put('errorMessage', e.getMessage());
        }
        return result;
    }

    /**
     * Method name : populateSObjectFields
     * @description : Method to populate the fields for creation of follow up objects
     * @return : void
     * @param : SObject newObj, Map<String, SObjectField> fieldMap, Map<String, Object> fieldValues
     */
    private static void populateSObjectFields(SObject newObj, Map<String, SObjectField> fieldMap, Map<String, Object> fieldValues) {
        for (String field : fieldValues.keySet()) {
            Object val = fieldValues.get(field);
            if (!fieldMap.containsKey(field)) {
                continue;
            }
            Schema.DisplayType type = fieldMap.get(field).getDescribe().getType();
            if (val != null) {
                switch on type {
                    when DATE {
                        if (val instanceof String) {
                            newObj.put(field, Date.valueOf((String)val));
                        } else {
                            newObj.put(field, val);
                        }
                    }
                    when DATETIME {
                        if (val instanceof String) {
                            newObj.put(field, DateTime.valueOf((String)val));
                        } else {
                            newObj.put(field, val);
                        }
                    }
                    when BOOLEAN {
                        if (val instanceof String) {
                            newObj.put(field, ((String)val).toLowerCase() == 'true');
                        } else if (val instanceof Boolean) {
                            newObj.put(field, val);
                        }
                    }
                    when INTEGER, DOUBLE, CURRENCY, PERCENT {
                        if (val instanceof String) {
                            newObj.put(field, Decimal.valueOf((String)val));
                        } else {
                            newObj.put(field, val);
                        }
                    }
                    when else {
                        newObj.put(field, val);
                    }
                }
            }
        }
    }

    /**
     * Method name : setOpportunityDefaults
     * @description : Method to setOpportunityDefaults for Opportunity object
     * @return : void
     * @param : SObject newObj, Map<String, SObjectField> fieldMap, Map<String, Object> fieldValues
     */
    private static void setOpportunityDefaults(SObject newObj, Map<String, SObjectField> fieldMap, Map<String, Object> fieldValues) {
        if (!fieldValues.containsKey(OWNER_ID) && fieldMap.containsKey(OWNER_ID)) {
            newObj.put(OWNER_ID, UserInfo.getUserId());
        }
        if (!fieldValues.containsKey(STAGE_NAME) && fieldMap.containsKey(STAGE_NAME)) {
            newObj.put(STAGE_NAME, STAGE_PRE_DISCOVERY);
        }
        if (!fieldValues.containsKey(LEGACY_ID) && fieldMap.containsKey(LEGACY_ID)) {
            newObj.put(LEGACY_ID, LEGACY_ID_VAL);
        }
        if (!fieldValues.containsKey(LEAD_SOURCE) && fieldMap.containsKey(LEAD_SOURCE)) {
            newObj.put(LEAD_SOURCE, LEAD_SOURCE_OTHER);
        }
        if ((!fieldValues.containsKey(CLOSE_DATE) || fieldValues.get(CLOSE_DATE) == null) 
            && fieldMap.containsKey(CLOSE_DATE)) {
            newObj.put(CLOSE_DATE, Date.today().addDays(90));
        }
    }

    /**
     * Method name : setTaskDefaults
     * @description : Method to setTaskDefaults for Task object
     * @return : void
     * @param : SObject newObj, Map<String, SObjectField> fieldMap, Map<String, Object> fieldValues
     */
    private static void setTaskDefaults(SObject newObj, Map<String, SObjectField> fieldMap, Map<String, Object> fieldValues, String taskId) {
        if (!fieldValues.containsKey(RECORD_TYPE_ID) && fieldMap.containsKey(RECORD_TYPE_ID)) {
            RecordType rt = [SELECT Id FROM RecordType WHERE SObjectType = :TASK AND DeveloperName = :TASK_DEV_NAME LIMIT 1];
            newObj.put(RECORD_TYPE_ID, rt.Id);
        }
        if (!fieldValues.containsKey(TYPE) && fieldMap.containsKey(TYPE)) {
            newObj.put(TYPE, TYPE_TODO);
        }
        if (!fieldValues.containsKey(STATUS) && fieldMap.containsKey(STATUS)) {
            newObj.put(STATUS, STATUS_NOT_STARTED);
        }
        if (!fieldValues.containsKey(LEGACY_ID) && fieldMap.containsKey(LEGACY_ID)) {
            newObj.put(LEGACY_ID, LEGACY_ID_VAL);
        }
        if (!fieldValues.containsKey(LINE_OF_BUSINESS) && fieldMap.containsKey(LINE_OF_BUSINESS)) {
            newObj.put(LINE_OF_BUSINESS, LINE_OF_BUSINESS_VALUE);
        }
        if (!fieldValues.containsKey(START_DATE) && fieldMap.containsKey(START_DATE)) {
            newObj.put(START_DATE, Date.today());
        }
    }

    /**
     * Method name : cleanupActionAndActionData
     * @description : Method for Linkage Between Proposed Follow Up Actions and Created Follow Up Actions
     * @return : void
     * @param : String actionId, Map<String, Object> result, fieldValues
     */
    private static void cleanupActionAndActionData(String actionId, Map<String, Object> result, Map<String, Object> fieldValues) {
        try {
            AF_MeetingNote_Action__c actionToUpdate = [SELECT Id, AF_Status__c, AF_Created_Follow_Up_Action__c, AF_Type__c
                FROM AF_MeetingNote_Action__c
                WHERE Id = :actionId
                LIMIT 1];
            
            actionToUpdate.AF_Status__c = 'Saved Successfully';
            actionToUpdate.AF_Created_Follow_Up_Action__c = (String) result.get('recordId');
            String createdType = (String) result.get('objectLabel');
            if (!String.isBlank(createdType)) {
                actionToUpdate.AF_Type__c = createdType;
            }
            update actionToUpdate;
        } catch (Exception deleteEx) {
            result.put('deleteWarning', deleteEx.getMessage());
        }
    }

    /**
     * Method name : getPickListValues
     * @description : Method to retrieve the picklist values
     * @return : Map<String, List<String>>
     * @param : None
     */
    @AuraEnabled(cacheable=true)
    public static Map<String, List<String>> getPicklists() {
        Map<String, List<String>> response = new Map<String, List<String>>();

        response.put(STATUS, getPicklist(TASK, STATUS));
        response.put(PRIORITY, getPicklist(TASK, PRIORITY));
        response.put(TYPE, getPicklist(TASK, TYPE));
        return response;
    }

    /**
     * Method name : getPicklist
     * @description : Method to populate the picklist values
     * @return : List<String>
     * @param : String objectName, String fieldName
     */
    private static List<String> getPicklist(String objectName, String fieldName) {
        Schema.DescribeFieldResult dfr = Schema.getGlobalDescribe().get(objectName).getDescribe().fields.getMap().get(fieldName).getDescribe();
        List<String> values = new List<String>();
        for (Schema.PicklistEntry e : dfr.getPicklistValues()) {
            if (e.isActive()) { values.add(e.getValue()); }
        }
        return values;
    }
}
