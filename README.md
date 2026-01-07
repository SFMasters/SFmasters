RecordType advisoryRT = [SELECT Id FROM RecordType WHERE SObjectType = 'Account' AND Name = 'Advisory Individual' LIMIT 1];

Account advisoryAccount = new Account(
    Name = 'Advisory Individual Account',
    RecordTypeId = advisoryRT.Id
);
insert advisoryAccount;
