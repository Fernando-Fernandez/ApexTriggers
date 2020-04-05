# ApexTriggers
Different types of Apex triggers (Salesforce) and their helper classes

All triggers call static methods on the helper class and follow the format below:

     trigger MyObjTrigger on ( before insert, before update, /* ... */ ) {
          MyObjHelperClass.myObjMethod( trigger.operationType, trigger.new, trigger.oldMap );
     }

All helper classes have a static method and a static set of ids to control recursion.
