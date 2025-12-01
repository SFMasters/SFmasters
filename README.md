/******************************************************************************************
 * @Description: * Creates temporary objects-AF_Meeting_Note_Action__c,AF_Meeting_Note_Action_Data__c
 * @Author: Sujaanaa (Deloitte USI)
 * @Date : 24-Oct-2025
 * @Test Class: AF_MeetingNoteActionCreator_Test
 * ******************************************************************************************
 * ModificationLog      Developer           CodeReview          Date            Description
 * 1.0                  Sujaanaa                                                
 * ******************************************************************************************/
global with sharing class AF_MeetingNoteActionCreator {

    public static final String REVIEW_STATUS = 'Requires Review';
    public static final String ACTIVITY_DATE = 'ActivityDate';

    //Wrapper for request details to send to agent
    global class Request {
        @InvocableVariable(required=true)
        global String objectApiName;
        @InvocableVariable(required=true)
        global String taskId;
        @InvocableVariable(required=false)
        global String relatedRecordId;
        @InvocableVariable(required=true)
        global String fieldApiNameToValueJSON;
        @InvocableVariable(required=true)
        global String actionName;
    }

    //Wrapper for storing reponse details from the agent 
    global class Response {
        @InvocableVariable global String message;
        @InvocableVariable global String tempActionId;
    }

    /**
     * Method name : createAction
     * @description : Method to Create Temporray Action object
     * @return : List<Response>
     * @param : List<Request> requests to create actions
     */
    @InvocableMethod(Label='Meeting Note Action Creator' description='Always creates new temp actions from a Meeting Note')
    global static List<Response> createAction(List<Request> requests) {
        System.debug('*** AF_MeetingNoteActionCreator.createAction - START ***');
        System.debug('Number of requests received: ' + requests.size());
        
        List<Response> responses = new List<Response>();
        List<AF_Meeting_Note_Action__c> actionsToInsert = new List<AF_Meeting_Note_Action__c>();
        Map<Integer, Request> requestByIndex = new Map<Integer, Request>();
        List<AF_Meeting_Note_Action_Data__c> actionDataToInsert = new List<AF_Meeting_Note_Action_Data__c>();
        List<Response> tempResponses = new List<Response>();
        Integer index = 0;

        try {
            // Get source task information to determine filtering logic
            Set<String> taskIds = new Set<String>();
            for (Request request : requests) {
                System.debug('Processing request - ObjectApiName: ' + request.objectApiName + ', TaskId: ' + request.taskId);
                if (String.isNotBlank(request.taskId)) {
                    taskIds.add(request.taskId);
                }
            }
            System.debug('Unique Task IDs collected: ' + taskIds);
            
            // Query source tasks to get WhoId and WhatId information
            Map<String, Task> sourceTaskMap = new Map<String, Task>();
            if (!taskIds.isEmpty()) {
                for (Task task : [SELECT Id, WhoId, WhatId 
                                FROM Task 
                                WHERE Id IN :taskIds 
                                LIMIT 1000]) {
                    sourceTaskMap.put(task.Id, task);
                    System.debug('Found Task - ID: ' + task.Id + ', WhoId: ' + task.WhoId + ', WhatId: ' + task.WhatId);
                }
            }
            System.debug('Total Tasks queried: ' + sourceTaskMap.size());
            
            //Build AF_Meeting_Note_Action__c records from requests with filtering logic
            for (Request request : requests) {
                System.debug('--- Processing Request ---');
                System.debug('ObjectApiName: ' + request.objectApiName);
                System.debug('TaskId: ' + request.taskId);
                
                boolean shouldCreateAction = true;
                
                // Check if this is an Opportunity request that should be filtered
                if (String.isNotBlank(request.objectApiName) && 
                    request.objectApiName.equalsIgnoreCase('Opportunity')) {
                    
                    System.debug('This is an Opportunity request - checking filtering conditions');
                    
                    Task sourceTask = sourceTaskMap.get(request.taskId);
                    System.debug('Source Task found: ' + (sourceTask != null));
                    
                    if (sourceTask != null && sourceTask.WhoId != null) {
                        String whoIdString = String.valueOf(sourceTask.WhoId);
                        System.debug('WhoId: ' + whoIdString + ', WhatId: ' + sourceTask.WhatId);
                        
                        // Check for Contact-only scenario: WhoId starts with '003' and WhatId is null
                        boolean isContactOnly = (whoIdString.startsWith('003') && sourceTask.WhatId == null);
                        System.debug('Is Contact-only scenario: ' + isContactOnly);
                        
                        // Check for Lead-only scenario: WhoId starts with '00Q' and WhatId is null
                        boolean isLeadOnly = (whoIdString.startsWith('00Q') && sourceTask.WhatId == null);
                        System.debug('Is Lead-only scenario: ' + isLeadOnly);
                        
                        // Don't create Opportunity actions for Contact-only or Lead-only meeting notes
                        if (isContactOnly || isLeadOnly) {
                            shouldCreateAction = false;
                            String recordType = isContactOnly ? 'Contact' : 'Lead';
                            System.debug('*** FILTERING OUT OPPORTUNITY *** - ' + recordType + '-only meeting note. Task ID: ' + sourceTask.Id);
                        } else {
                            System.debug('Opportunity request ALLOWED - not Contact-only or Lead-only scenario');
                        }
                    } else {
                        System.debug('Source Task not found or WhoId is null - allowing Opportunity request');
                    }
                } else {
                    System.debug('Not an Opportunity request - proceeding normally');
                }
                
                System.debug('Final decision - shouldCreateAction: ' + shouldCreateAction);
                
                if (shouldCreateAction) {
                    System.debug('CREATING ACTION for ObjectApiName: ' + request.objectApiName);
                    AF_Meeting_Note_Action__c action = new AF_Meeting_Note_Action__c(
                        AF_Record_ID__c     = request.relatedRecordId,
                        AF_Object_Name__c   = request.objectApiName,
                        AF_Status__c        = REVIEW_STATUS,
                        AF_Action_Name__c   = request.actionName,
                        AF_Meeting_Note__c  = request.taskId
                    );
                    actionsToInsert.add(action);
                    requestByIndex.put(index, request);
                    System.debug('Added action to insert list. Current actionsToInsert size: ' + actionsToInsert.size());
                    index++;
                } else {
                    System.debug('*** NOT CREATING ACTION *** for ObjectApiName: ' + request.objectApiName + ' - FILTERED OUT');
                    // Create a response for the filtered request
                    Response filteredResponse = new Response();
                    filteredResponse.message = 'Opportunity creation not allowed for Contact-only or Lead-only meeting notes';
                    filteredResponse.tempActionId = null;
                    responses.add(filteredResponse);
                    System.debug('Added filtered response. Current responses size: ' + responses.size());
                }
            }
            
            System.debug('Total actions to insert: ' + actionsToInsert.size());
            System.debug('Actions to insert details:');
            for(AF_Meeting_Note_Action__c action : actionsToInsert) {
                System.debug('- Action ObjectType: ' + action.AF_Object_Name__c + ', ActionName: ' + action.AF_Action_Name__c);
            }
            
            if (!actionsToInsert.isEmpty()) {
                System.debug('Performing bulk insert of ' + actionsToInsert.size() + ' actions');
                insert actionsToInsert;
                System.debug('Bulk insert completed successfully');
            } else {
                System.debug('*** NO ACTIONS TO INSERT - All were filtered out ***');
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
                            //try {
                                dueDate = Date.valueOf(dueDateStr);
                            //} catch (Exception e) {
                                //dueDate = Date.today();
                            //}
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
        return responses;
    }
}
