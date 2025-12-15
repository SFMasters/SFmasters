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
                    
                    // Check if this is Product Group field on Opportunity - use Custom Labels approach
                    if (objectName == 'Opportunity' && caseSensitiveFieldName == 'Product_Group__c') {
                        try {
                            // CUSTOM LABELS APPROACH - Restrict agent to only allowed values (e.g., 15 out of 30 total)
                            String labelValues = System.Label.Financial_Solutions_Product_Groups;
                            if (String.isNotBlank(labelValues)) {
                                // Split comma-separated values from Custom Label
                                List<String> allowedValues = labelValues.split(',');
                                // Trim whitespace from each value (handles spacing after commas)
                                for (String value : allowedValues) {
                                    picklistValues.add(value.trim());
                                }
                                System.debug('Product_Group__c restricted to Custom Label values: ' + picklistValues.size() + ' out of total available');
                                System.debug('Allowed values: ' + picklistValues);
                            }
                            // No else block - if Custom Label is empty, agent gets no values (intentional restriction)
                        } catch (Exception e) {
                            System.debug('Error filtering Product_Group__c values with Custom Labels: ' + e.getMessage());
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
