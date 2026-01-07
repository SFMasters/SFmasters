@isTest
static void testUS58_PersonLifeEvent_AllFilteringScenarios() {
    Test.startTest();
    
    // Create test user
    User testUser = AF_TestDataFactory.createUser(new Map<String, Object>{'LastName' => 'US58 User'});
    
    // Assign permission set for Account access
    PermissionSet ps = [SELECT Id FROM PermissionSet 
                        WHERE Name LIKE '%Account%' OR Name LIKE '%Sales%' 
                        LIMIT 1];
    PermissionSetAssignment psa = new PermissionSetAssignment(
        PermissionSetId = ps.Id,
        AssigneeId = testUser.Id
    );
    insert psa;
    
    AF_TestDataFactory.createInvocationRecord(testUser.Id);

    System.runAs(testUser) {
        // Now Account creation should work with proper permissions
        Account acc = AF_TestDataFactory.createAccount(new Map<String, Object>{'Name' => 'US58 Test Account'});
        
        RecordType advisoryRT = [SELECT Id FROM RecordType WHERE SObjectType = 'Account' AND Name = 'Advisory Individual' LIMIT 1];
        Account advisoryAccount = new Account(
            Name = 'Advisory Individual Account',
            RecordTypeId = advisoryRT.Id
        );
        insert advisoryAccount; // Should work now!
        
        // ... rest of test method
    }
}
