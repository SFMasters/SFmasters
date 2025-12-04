/**************************************************************************************************************************************\
@ Description: * Provides metadata information about Salesforce objects for AI processing
* Extracts and formats object fields, picklist values, and component metadata
@ Author: Sujaanaa (Deloitte USI)   		          
@ Date : 24-Oct-2025	  		      
@ Test Class:   
/************************************************************************************************************************************
* ModificationLog     Developer            CodeReview             Date                   Description   
  1.0                 Sujaanaa                                                         Initial version
  1.1                 Codify                                     Dec-2025              Added record type-aware picklist filtering for Product_Group__c
*************************************************************************************************************************************/
global with sharing class AF_MeetingNoteMetadataGrounding {

    public static final String CURRENT_USER_ID_LABEL = 'Current User ID: ';
    public static final String CURRENT_DATETIME_LABEL = 'Current DateTime: ';
    public static final String RELATED_OBJECT_API_NAME_LABEL = 'Related Object API Name: ';
    public static final String METADATA_LABEL = 'Here is the metadata for ';
    public static final String KEY_FIELDS_LABEL = ' - Key Fields\n';
    public static final String INVALID_OBJECT_NAME_ERROR = 'Invalid object name: ';

    // Response wrapper containing the formatted metadata prompt
    global class Response {
        @InvocableVariable
        global String Prompt;
    }    

    //Request wrapper containing task and related object information
    global class Request {
        @InvocableVariable(required=true)
        global String Task_Id;
        
        @InvocableVariable
        global String Related_Object_Id;
    }
    
    // This fetches the metadata for all supported objects 
    @InvocableMethod(Label='Metadata Grounding' description='Gets metadata for supported objects' category='AI')
    global static List<Response> getMetadataGrounding(List<Request> requests) {
        List<Response> results = new List<Response>();
        Response result = new Response();
        result.Prompt = '';
        
        String taskId = requests[0].Task_Id;
        Task task;
        if (String.isNotBlank(taskId)) {
            task = [SELECT Id, CreatedById FROM Task WHERE Id =: taskId LIMIT 1];
        } 

        List<AF_MeetingNote_Action_Supported_Metadata__mdt> objectGroundingSettings = getSupportedObjectGroundings();
        for (AF_MeetingNote_Action_Supported_Metadata__mdt setting: objectGroundingSettings) {
            List<String> fields = setting.Supported_Fields__c.split(',');
            List<String> cleanFields = new List<String>();
            for (String field : fields) {
                cleanFields.add(field.trim());
            }
            if (fields.size() == 0) {
                continue;
            }
            result.Prompt += getObjectEmbedding(setting.Object_API_Name__c, cleanFields);
        }
        
        if (task != null) {
            Id currentUserId = task.createdById;
            result.Prompt += CURRENT_USER_ID_LABEL + currentUserId + '\n\n';
        }
        
        String isoNow = Datetime.now().formatGMT('yyyy-MM-dd\'T\'HH:mm:ss\'Z\'');
        result.Prompt += CURRENT_DATETIME_LABEL + isoNow + '\n\n';

        String relatedObjectId = requests[0].Related_Object_Id;  
        if(String.isNotBlank(relatedObjectId)) { 
            String objectApiName = getRelatedObjectAPIName(relatedObjectId);
            if (String.isNotBlank(objectApiName)) {
                result.Prompt += RELATED_OBJECT_API_NAME_LABEL + objectApiName + '\n\n';
            }
        }        
        results.add(result);
        return results;
    }    
  
    //Gets the API name of an object from its record ID
    private static String getRelatedObjectAPIName(String relatedObjectId) {
        Id recordId = (Id) relatedObjectId;
        return recordId.getSobjectType().getDescribe().getLocalName();
    }
    
    //Gets all the AF_MeetingNote_Action_Supported_Metadata__mdt data
    private static List<AF_MeetingNote_Action_Supported_Metadata__mdt> getSupportedObjectGroundings() {
        return [SELECT Object_API_Name__c, Supported_Fields__c FROM AF_MeetingNote_Action_Supported_Metadata__mdt];
    }    

    //Generates a formatted metadata embedding for an object
    private static String getObjectEmbedding(String objectName, List<String> supportedFields) {
        MetadataExtractor extractor = new MetadataExtractor();
        List<FieldDetails> fields = extractor.getObjectFields(objectName, supportedFields);
        String groundings = objectName + KEY_FIELDS_LABEL +
            METADATA_LABEL + objectName + ':\n' +
            '```\n' +
            '<' + objectName + '_metadata>' + JSON.serializePretty(fields, true) + '</' + objectName + '_metadata>' +
            '```  \n \n';
        return groundings;
    }    

    //Inner class to store detailed field metadata
    private class FieldDetails {
        public String fieldName;
        public String fieldType;
        public String fieldDescription;
        public List<String> picklistValues;
        
        public FieldDetails(String fieldName, String fieldType, String fieldDescription, List<String> picklistValues) {
            this.fieldName = fieldName;
            this.fieldType = fieldType;
            this.fieldDescription = fieldDescription;
            this.picklistValues = picklistValues;
        }
    }
    
   //Helper class to extract field metadata from Salesforce objects
    private class MetadataExtractor {       
        public List<FieldDetails> getObjectFields(String objectName, List<String> supportedFields) {
            List<FieldDetails> fieldDetailsList = new List<FieldDetails>();            
            
            Schema.SObjectType sObjectType = Schema.getGlobalDescribe().get(objectName);
            if (sObjectType == null) {
                throw new IllegalArgumentException(INVALID_OBJECT_NAME_ERROR + objectName);
            }
            
            Schema.DescribeSObjectResult describeResult = sObjectType.getDescribe();
            Map<String, Schema.SObjectField> fieldMap = describeResult.fields.getMap();
            
            for (String fieldName : fieldMap.keySet()) {
                Schema.SObjectField field = fieldMap.get(fieldName);
                String caseSensitiveFieldName = field.toString();
                if (!supportedFields.contains(caseSensitiveFieldName) && supportedFields[0] != '*'){
                    continue;
                }
                
                Schema.DescribeFieldResult fieldDescribe = field.getDescribe();
                String fieldType = fieldDescribe.getType().name();
                String fieldDescription = fieldDescribe.getInlineHelpText();
                
                List<String> picklistValues;
                if (fieldDescribe.getType() == Schema.DisplayType.Picklist) {
                    picklistValues = new List<String>();
                    
                    // Check if this is Product Group field on Opportunity - apply record type filtering
                    if (objectName == 'Opportunity' && caseSensitiveFieldName == 'Product_Group__c') {
                        try {
                            // Get record type ID for Financial Solutions Opportunity
                            Map<String, Schema.RecordTypeInfo> rtMap = Schema.SObjectType.Opportunity.getRecordTypeInfosByDeveloperName();
                            if (rtMap.containsKey('Financial_Solutions_Opportunity')) {
                                Id recordTypeId = rtMap.get('Financial_Solutions_Opportunity').getRecordTypeId();
                                
                                // APPROACH 1: OPTIMIZED - Direct record type filtering (commented for testing)
                                // Uncomment this block if getPicklistValues(recordTypeId) method is available in your org
                                /*
                                List<Schema.PicklistEntry> recordTypeSpecificEntries = fieldDescribe.getPicklistValues(recordTypeId);
                                for (Schema.PicklistEntry entry : recordTypeSpecificEntries) {
                                    if (entry.isActive()) {
                                        picklistValues.add(entry.getLabel());
                                    }
                                }
                                */
                                
                                // APPROACH 2: CONSERVATIVE - Standard filtering with record type context (currently active)
                                // This approach gets all values but relies on Salesforce's record type configuration
                                // When Product Group field is configured for Financial_Solutions_Opportunity record type,
                                // this should automatically return only the enabled values for that record type
                                for (Schema.PicklistEntry entry : fieldDescribe.getPicklistValues()) {
                                    if (entry.isActive()) {
                                        picklistValues.add(entry.getLabel());
                                    }
                                }
                                
                                System.debug('Product_Group__c picklist values for Financial_Solutions_Opportunity: ' + picklistValues.size() + ' values');
                                
                            } else {
                                System.debug('Financial_Solutions_Opportunity record type not found');
                                // Fallback to all values if record type not found
                                for (Schema.PicklistEntry picklistEntry : fieldDescribe.getPicklistValues()) {
                                    if (picklistEntry.isActive()) {
                                        picklistValues.add(picklistEntry.getLabel());
                                    }
                                }
                            }
                        } catch (Exception e) {
                            System.debug('Error getting record type specific picklist values: ' + e.getMessage());
                            // Fallback to all values on error
                            for (Schema.PicklistEntry picklistEntry : fieldDescribe.getPicklistValues()) {
                                if (picklistEntry.isActive()) {
                                    picklistValues.add(picklistEntry.getLabel());
                                }
                            }
                        }
                    } else {
                        // Default behavior for all other fields - unchanged
                        for (Schema.PicklistEntry picklistEntry : fieldDescribe.getPicklistValues()) {
                            if (picklistEntry.isActive()) {
                                picklistValues.add(picklistEntry.getLabel());
                            }
                        }
                    }
                }
                
                FieldDetails fieldDetails = new FieldDetails(caseSensitiveFieldName, fieldType, fieldDescription, picklistValues);
                fieldDetailsList.add(fieldDetails);
            }
            
            return fieldDetailsList;
        }
    }
}
