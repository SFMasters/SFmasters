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
                if (String.isNotBlank(request.taskId)) {
                    taskIds.add(request.taskId);
                }
            }
            
            // Query source tasks to get WhoId and WhatId information
            Map<String, Task> sourceTaskMap = new Map<String, Task>();
            if (!taskIds.isEmpty()) {
                for (Task task : [SELECT Id, WhoId, WhatId 
                                FROM Task 
                                WHERE Id IN :taskIds 
                                LIMIT 1000]) {
                    sourceTaskMap.put(task.Id, task);
                }
            }
            
            //Build AF_Meeting_Note_Action__c records from requests with filtering logic
            for (Request request : requests) {
                boolean shouldCreateAction = true;
                
                // Check if this is an Opportunity request that should be filtered
                if (String.isNotBlank(request.objectApiName) && 
                    request.objectApiName.equalsIgnoreCase('Opportunity')) {
                    
                    Task sourceTask = sourceTaskMap.get(request.taskId);
                    if (sourceTask != null && sourceTask.WhoId != null) {
                        String whoIdString = String.valueOf(sourceTask.WhoId);
                        
                        // Check for Contact-only scenario: WhoId starts with '003' and WhatId is null
                        boolean isContactOnly = (whoIdString.startsWith('003') && sourceTask.WhatId == null);
                        
                        // Check for Lead-only scenario: WhoId starts with '00Q' and WhatId is null
                        boolean isLeadOnly = (whoIdString.startsWith('00Q') && sourceTask.WhatId == null);
                        
                        // Don't create Opportunity actions for Contact-only or Lead-only meeting notes
                        if (isContactOnly || isLeadOnly) {
                            shouldCreateAction = false;
                            String recordType = isContactOnly ? 'Contact' : 'Lead';
                            System.debug('Filtering out Opportunity request for ' + recordType + '-only meeting note. Task ID: ' + sourceTask.Id);
                        }
                    }
                }
                
                if (shouldCreateAction) {
                    AF_Meeting_Note_Action__c action = new AF_Meeting_Note_Action__c(
                        AF_Record_ID__c     = request.relatedRecordId,
                        AF_Object_Name__c   = request.objectApiName,
                        AF_Status__c        = REVIEW_STATUS,
                        AF_Action_Name__c   = request.actionName,
                        AF_Meeting_Note__c  = request.taskId
                    );
                    actionsToInsert.add(action);
                    requestByIndex.put(index, request);
                    index++;
                } else {
                    // Create a response for the filtered request
                    Response filteredResponse = new Response();
                    filteredResponse.message = 'Opportunity creation not allowed for Contact-only or Lead-only meeting notes';
                    filteredResponse.tempActionId = null;
                    responses.add(filteredResponse);
                }
            }
            
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
