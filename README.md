/****************************************************************************************
* @Description : Creates temporary objects AF_MeetingNote_Action__c, AF_MeetingNote_Action_Data__c
* @Author: Sujaanaa (Deloitte USI)
* @Date : 24-Oct-2025
* @Test Class: AF_MeetingNoteActionCreator_Test
*****************************************************************************************/
/* ModificationLog      Developer       CodeReview      Date            Description
* 1.0                  Sujaanaa                        24-Oct-2025     Initial Version
* 2.0                  Arya Shah                       2/12/2025       Prevent Opportunity creation for Contact-only or Lead-only meeting notes
*****************************************************************************************/
global without sharing class AF_MeetingNoteActionCreator {

    public static final String REVIEW_STATUS = 'Requires Review';
    public static final String ACTIVITY_DATE = 'ActivityDate';
    public static final String OBJECT_MILESTONE = 'PersonLifeEvent';
    public static final String OBJECT_OPPORTUNITY = 'Opportunity';

    public static final String SKIP_LEAD = 'Skipped: WhoId is a Lead. Action not created.';
    public static final String SKIP_MILESTONE = 'Skipped: PersonLifeEvent creation not allowed for Contact-only meeting notes';
    public static final String SKIP_OPPORTUNITY = 'Skipped: Opportunity creation not allowed for contact/ lead only meeting notes';

    public static final String SKIP_EVENT_TYPE_BLANK = 'Skipped: Event type is blank. Action not created.';
    public static final String SKIP_MILESTONE_EXIST = 'Skipped: PersonLifeEvent already exists for this person, type, and date.';
    // global static boolean setFromLWC;

    //Wrapper for request details to send to agent
    global class Request {
        @InvocableVariable(required=true)
        global String objectApiName;
        @InvocableVariable(required=true)
        global String fieldApiNameToValueJSON;
        @InvocableVariable(required=true)
        global String taskId;
        @InvocableVariable(required=false)
        global String relatedRecordId;
        @InvocableVariable(required=true)
        global String actionName;
    }

    //Wrapper for storing response details from the agent
    global class Response {
        @InvocableVariable global String message;
        @InvocableVariable global String tempActionId;
    }

    /**
    * Method name : createAction
    * @description : Method to Create Temporary Action object
    * @return : List<Response>
    * @param : List<Request> requests to create actions
    */
    @InvocableMethod(Label='Meeting Note Action Creator' description='Always creates new temp actions from a Meeting Note')
    global static List<Response> createAction(List<Request> requests) {

        List<Response> responses = new List<Response>();
        List<AF_Meeting_Note_Action__c> actionsToInsert = new List<AF_Meeting_Note_Action__c>();
        Map<Integer, Request> requestByIndex = new Map<Integer, Request>();
        List<AF_Meeting_Note_Action_Data__c> actionDataToInsert = new List<AF_Meeting_Note_Action_Data__c>();
        List<Response> tempResponses = new List<Response>();
        Integer index = 0;

        List<AF_Invocation__c> invocationRecordList = getInvocationRecord();
        deleteInvocationRecords();
        if(!invocationRecordList.isEmpty()){
            try {
                Set<Id> taskIds = new Set<Id>();
                for(Request request : requests) {
                    if(String.isNotBlank(request.taskId)) {
                        taskIds.add(request.taskId);
                    }
                }

                Map<Id, Task> sourceTaskMap = new Map<Id, Task>();
                if(!taskIds.isEmpty()) {
                    // Calculate dynamic limit based on remaining query rows to avoid governor limits
                    Integer remainingQueryRows = Limits.getLimitQueryRows() - Limits.getQueryRows();
                    Integer queryLimit = Math.min(remainingQueryRows, taskIds.size());

                    if (queryLimit > 0) {
                        for (Task task : [SELECT Id, WhoId, WhatId
                                        FROM Task
                                        WHERE Id IN :taskIds
                                        LIMIT :queryLimit]) {
                            sourceTaskMap.put(task.Id, task);
                            System.debug('Found Task - ID: ' + task.Id + ', WhoId: ' + task.WhoId + ', WhatId: ' + task.WhatId);
                        }
                    } else {
                        System.debug('WARNING: No query rows remaining. Cannot query tasks for filtering logic.');
                    }
                }
                System.debug('Total Tasks queried: ' + sourceTaskMap.size());

                for(Request request : requests) {
                    Boolean skip = false;
                    String skipMessage = '';
                    String eventType = '';
                    String eventDateStr = '';
                    String whoId = null;

                    if (sourceTaskMap.containsKey(request.taskId)) {
                        Task sourceTask = sourceTaskMap.get(request.taskId);
                        whoId = (sourceTask.WhoId != null) ? String.valueOf(sourceTask.WhoId) : null;

                        // Business Rule: Prevent Opportunity creation for Contact-only or Lead-only meeting notes
                        if (request.objectApiName == OBJECT_OPPORTUNITY) {
                            if (sourceTask != null && whoId != null) {
                                // Schema-based Contact detection and Lead detection
                                boolean isContactOnly = (Id.valueOf(whoId).getSObjectType() == Contact.SObjectType && sourceTask.WhatId == null);
                                boolean isLeadOnly = (whoId.startsWith('00Q') && sourceTask.WhatId == null);
                                
                                if (isContactOnly || isLeadOnly) {
                                    skip = true;
                                    skipMessage = SKIP_OPPORTUNITY;
                                    System.debug('*** SKIPPING OPPORTUNITY *** - Contact/Lead-only meeting note. Task ID: ' + sourceTask.Id);
                                }
                            }
                        }
                        
                        // US-58: PersonLifeEvent filtering logic
                        if (request.objectApiName == OBJECT_MILESTONE) {
                            // Always skip for Lead WhoId or null WhoId (regardless of WhatId)
                            if ((whoId != null && whoId.startsWith('00Q')) || whoId == null) {
                                skip = true;
                                skipMessage = SKIP_LEAD;
                                System.debug('PersonLifeEvent skipMessage: ' + skipMessage);
                            } else if (whoId != null && Id.valueOf(whoId).getSObjectType() == Contact.SObjectType) {
                                // Contact logic: Check WhatId for Contact-only scenario
                                if (sourceTask != null && sourceTask.WhatId == null) {
                                    // Contact-only scenario
                                    skip = true;
                                    skipMessage = SKIP_MILESTONE;
                                    System.debug('*** SKIPPING PERSONLIFEEVENT *** - Contact-only meeting note. Task ID: ' + sourceTask.Id);
                                } else {
                                    // Contact with WhatId - proceed with duplicate check
                                    List<Object> fieldList = (List<Object>) JSON.deserializeUntyped(request.fieldApiNameToValueJSON);
                                    for (Object field : fieldList) {
                                        Map<String, Object> fieldMap = (Map<String, Object>)field;
                                        if (fieldMap.containsKey('EventType')) {
                                            eventType = String.valueOf(fieldMap.get('EventType'));
                                        }
                                        if (fieldMap.containsKey('EventDate')) {
                                            eventDateStr = String.valueOf(fieldMap.get('EventDate'));
                                        }
                                    }

                                    // Duplicate check for existing PersonLifeEvent
                                    if (eventType != '' && eventDateStr != '') {
                                        Date eventDate;
                                        try { 
                                            eventDate = Date.valueOf(eventDateStr); 
                                        } catch(Exception e) { 
                                            eventDate = null; 
                                        }
                                        if (eventDate != null) {
                                            List<PersonLifeEvent> foundEvents = [
                                                SELECT Id FROM PersonLifeEvent
                                                WHERE PrimaryPersonId = :whoId
                                                  AND EventType = :eventType
                                                  AND EventDate = :eventDate
                                                LIMIT 1
                                            ];
                                            if (!foundEvents.isEmpty()) {
                                                skip = true;
                                                skipMessage = SKIP_MILESTONE_EXIST;
                                                System.debug('*** SKIPPING PERSONLIFEEVENT *** - Duplicate found.');
                                            }
                                        }
                                    }
                                }
                            }
                        }
                    }

                    if (skip) {
                        Response resp = new Response();
                        resp.message = skipMessage;
                        resp.tempActionId = null;
                        responses.add(resp);
                        continue;
                    } 
                    
                    // Code for creating AF_Meeting_Note_Action__c action object and adding to actionsToInsert list
                    AF_Meeting_Note_Action__c action = new AF_Meeting_Note_Action__c(
                        AF_Record_ID__c = request.relatedRecordId,
                        AF_Object_Name__c = request.objectApiName,
                        AF_Status__c = REVIEW_STATUS,
                        AF_Action_Name__c = request.actionName,
                        AF_Meeting_Note__c = request.taskId
                    );
                    actionsToInsert.add(action);
                    requestByIndex.put(index, request);
                    index++;
                }
                
                System.debug('currentTime'+DateTime.now());
                if (!actionsToInsert.isEmpty()) {
                    insert actionsToInsert;
                }
            } catch (Exception ex) {
                for (Request req : requests) {
                    Response errorResponse = new Response();
                    errorResponse.message = 'Error creating temp actions: ' + ex.getMessage();
                    errorResponse.tempActionId = null;
                    responses.add(errorResponse);
                }
                return responses;
            }

            // Process each request to build corresponding Action_Data__c records
            for (Integer i = 0; i < actionsToInsert.size(); i++) {
                AF_Meeting_Note_Action__c action = actionsToInsert[i];
                Request request = requestByIndex.get(i);
                Response response = new Response();
                response.tempActionId = action.Id;
                try {
                    List<Object> fieldApiNameToValueList = (List<Object>) JSON.deserializeUntyped(request.fieldApiNameToValueJSON);
                    for (Object field : fieldApiNameToValueList) {
                        Map<String, Object> fieldObject = (Map<String, Object>) field;
                        for (String key : fieldObject.keySet()) {
                            AF_Meeting_Note_Action_Data__c data = new AF_Meeting_Note_Action_Data__c();
                            data.AF_Field_Name__c = key;

                            // Due Date logic: check for past and set to today if required
                            if (key == ACTIVITY_DATE) {
                                String dueDateStr = String.valueOf(fieldObject.get(key));
                                Date dueDate;
                                dueDate = Date.valueOf(dueDateStr);
                                if (dueDate < Date.today()) {
                                    data.AF_Field_Value__c = String.valueOf(Date.today());
                                } else {
                                    data.AF_Field_Value__c = String.valueOf(dueDate);
                                }
                            } else {
                                data.AF_Field_Value__c = String.valueOf(fieldObject.get(key));
                            }
                            data.AF_Meeting_Note_Action__c = action.Id;
                            actionDataToInsert.add(data);
                        }
                    }
                    response.message = 'Successfully created the temp action!';
                } catch (Exception dataEx) {
                    response.message = 'Error processing field  ' + dataEx.getMessage();
                }
                tempResponses.add(response);
            }

            // Bulk insert of Action_Data records
            if (!actionDataToInsert.isEmpty()) {
                try {
                    insert actionDataToInsert;
                } catch (Exception dataEx) {
                    for (Response resp : tempResponses) {
                        if(String.isEmpty(resp.message)){
                            resp.message = 'Partial failure: Action Data insert failed: ' + dataEx.getMessage();
                        }
                    }
                }
            }
            
            responses.addAll(tempResponses);

        } else {
            Response nonLWCResponse = new Response();
            nonLWCResponse.message = 'I am not able to perform this operation';
            nonLWCResponse.tempActionId = null;
            responses.add(nonLWCResponse);
        }
        
        return responses;
    }

    /**
    * Method name : getInvocationRecord
    * @description : Method to get Invocation Records
    * @return : List<AF_Invocation__c>
    * @param : Nil
    */
    public static List<AF_Invocation__c> getInvocationRecord(){
        DateTime checkTime = DateTime.now().addSeconds(-20);
        System.debug('checkTime'+checkTime);
        List<AF_Invocation__c> recentInvocations = [SELECT Id, AF_Current_Timestamp__c FROM AF_Invocation__c
                                                    WHERE AF_User_ID__c = :UserInfo.getUserId()
                                                    AND AF_Current_Timestamp__c >= :checkTime
                                                    ORDER BY AF_Current_Timestamp__c DESC
                                                    LIMIT 1];
        System.debug('recentInvocations'+recentInvocations);
        return recentInvocations;
    }

    /**
    * Method name : deleteInvocationRecords
    * @description : Method to delete Invocation Records
    * @return : Void
    * @param : Nil
    */
    public static void deleteInvocationRecords(){
        List<AF_Invocation__c> oldInvocations = [SELECT Id, AF_Current_Timestamp__c FROM AF_Invocation__c
                                                 WHERE AF_Current_Timestamp__c <= :System.now().addMinutes(-5)];
        System.debug('oldInvocations'+oldInvocations);
        delete oldInvocations;
    }
}
