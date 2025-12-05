Remote Site URL:
https://.*\.my\.salesforce\.com

Remote Site Setting to allow HTTP callouts to current org for UI API access

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
                    
                    // Check if this is Product Group field on Opportunity - apply UI API dynamic filtering
                    if (objectName == 'Opportunity' && caseSensitiveFieldName == 'Product_Group__c') {
                        try {
                            // Find the Record Type ID by developer name
                            Id recordTypeId = null;
                            for (Schema.RecordTypeInfo rtInfo : describeResult.getRecordTypeInfos()) {
                                if (rtInfo.getDeveloperName() == 'Financial_Solutions_Opportunity') {
                                    recordTypeId = rtInfo.getRecordTypeId();
                                    break;
                                }
                            }
                            
                            if (recordTypeId != null) {
                                // DYNAMIC UI API APPROACH - Get record type specific picklist values via HTTP
                                List<String> dynamicValues = getRecordTypePicklistValues(objectName, caseSensitiveFieldName, recordTypeId);
                                
                                if (!dynamicValues.isEmpty()) {
                                    // Use dynamic values from UI API
                                    picklistValues.addAll(dynamicValues);
                                    System.debug('Product_Group__c filtered using UI API: ' + picklistValues.size() + ' dynamic values');
                                    System.debug('Record Type ID: ' + recordTypeId);
                                    System.debug('Dynamic values: ' + picklistValues);
                                } else {
                                    System.debug('UI API returned no values, using fallback');
                                    // Fallback to all values if UI API returns nothing
                                    for (Schema.PicklistEntry entry : fieldDescribe.getPicklistValues()) {
                                        if (entry.isActive()) {
                                            picklistValues.add(entry.getLabel());
                                        }
                                    }
                                }
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
                            System.debug('Error filtering Product_Group__c values with UI API: ' + e.getMessage());
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
    
    /**
     * Method name      : getRecordTypePicklistValues
     * @description     : Gets picklist values for specific record type using UI API
     * @return          : List<String>
     * @param           : String objectName, String fieldName, Id recordTypeId
     */
    private static List<String> getRecordTypePicklistValues(String objectName, String fieldName, Id recordTypeId) {
        List<String> recordTypeValues = new List<String>();
        
        try {
            // Construct UI API endpoint for record type specific picklist values
            String endpoint = URL.getOrgDomainUrl().toExternalForm() + 
                            '/services/data/v58.0/ui-api/object-info/' + objectName + 
                            '/picklist-values/' + recordTypeId + '/' + fieldName;
            
            // Create HTTP request
            HttpRequest req = new HttpRequest();
            req.setEndpoint(endpoint);
            req.setHeader('Authorization', 'Bearer ' + UserInfo.getSessionId());
            req.setHeader('Content-Type', 'application/json');
            req.setMethod('GET');
            req.setTimeout(10000); // 10 second timeout
            
            // Make the callout
            Http http = new Http();
            HttpResponse res = http.send(req);
            
            System.debug('UI API Endpoint: ' + endpoint);
            System.debug('UI API Response Status: ' + res.getStatusCode());
            System.debug('UI API Response Body: ' + res.getBody());
            
            if (res.getStatusCode() == 200) {
                // Parse the JSON response
                Map<String, Object> responseBody = (Map<String, Object>) JSON.deserializeUntyped(res.getBody());
                
                // Navigate to the values section
                if (responseBody.containsKey('values')) {
                    Map<String, Object> values = (Map<String, Object>) responseBody.get('values');
                    
                    // Extract picklist values
                    for (String valueKey : values.keySet()) {
                        Map<String, Object> valueInfo = (Map<String, Object>) values.get(valueKey);
                        
                        // Check if this value is valid for the record type
                        if (valueInfo.containsKey('validFor') && valueInfo.containsKey('label')) {
                            List<Object> validForList = (List<Object>) valueInfo.get('validFor');
                            String label = (String) valueInfo.get('label');
                            
                            // Check if current record type is in the validFor list
                            if (validForList != null && validForList.contains(recordTypeId)) {
                                recordTypeValues.add(label);
                            }
                        }
                    }
                }
                
                System.debug('UI API extracted ' + recordTypeValues.size() + ' values: ' + recordTypeValues);
                
            } else {
                System.debug('UI API call failed with status: ' + res.getStatusCode() + ', Body: ' + res.getBody());
            }
            
        } catch (CalloutException e) {
            System.debug('UI API Callout Exception: ' + e.getMessage());
        } catch (Exception e) {
            System.debug('UI API General Exception: ' + e.getMessage());
        }
        
        return recordTypeValues;
    }
