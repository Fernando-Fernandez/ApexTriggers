# ApexTriggers
Different types of Apex triggers (Salesforce) and their helper classes

All triggers call static methods on the helper class and follow the format below:

     trigger MyObjTrigger on ( before insert, before update, /* ... */ ) {
          MyObjHelperClass.myObjMethod( trigger.operationType, trigger.new, trigger.oldMap );
     }

All helper classes have a static method and a static set of ids to control recursion.

## Types of triggers
There are 6 most basic types of triggers (there may be others but these are the ones most common):

* Trigger sets fields with values derived from other fields in the same record

* Trigger adds validation error messages using data from fields in the same record

* Trigger sets fields with values from other records

* Trigger sets fields with values on other records

* Trigger performs call outs to external web services or HTTP requests to receive data

* Trigger sends an email
