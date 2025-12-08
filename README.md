 /**
     * Method name      : getRecordTypePicklistValues
     * @description     : Gets picklist values for specific record type using UI API
     * @return          : List<String>
     * @param           : String objectName, String fieldName, Id recordTypeId
     */
    private static List<String> getRecordTypePicklistValues(String objectName, String fieldName, Id recordTypeId) {
        List<String> recordTypeValues = new List<String>();
        
        try {
            // Use Named Credential endpoint - handles authentication automatically
            String endpoint = 'callout:AF_Outh/services/data/v58.0/ui-api/object-info/' + objectName + 
                            '/picklist-values/' + recordTypeId + '/' + fieldName;
            
            // Create HTTP request - no manual authentication needed
            HttpRequest req = new HttpRequest();
            req.setEndpoint(endpoint);
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
