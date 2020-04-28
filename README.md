# ApexTriggers
Different types of Apex triggers (Salesforce) and their helper classes

All triggers call static methods on the helper class and follow the format below:

     trigger MyObjTrigger on MyObj__c ( before insert, before update, /* ... */ ) {
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

### Trigger sets fields with values derived from other fields in the same record
This trigger does what a formula field would ideally do:  compute an expression using other field values. 
Some expressions are not possible using formula functions or the resulting expression exceeds the formula size limit, hence the requirement for this trigger. 
For example, there is no formula field that calculates the internal rate of return ([IRR or XIRR](https://en.wikipedia.org/wiki/Internal_rate_of_return)), which can only be calculated using Newton's iterative method.

     trigger ScenarioTrigger on Scenario__c ( before insert, before update ) {
          ScenarioHelperClass.calculateXIRR( trigger.operationType, trigger.new, trigger.oldMap );
     }
     
     class ScenarioHelperClass {
          public static void calculateXIRR( TriggerOperation operationType
                              , List<Scenario__c> newList, Map<ID, Scenario__c> oldMap ) {
               // TODO:  implement recursion prevention here
               
               XIRRHelper myXIRR = new XIRRHelper();
               
               for( Scenario__c aScenario : newList ) {
                    // TODO:  implement validation of periods and cashflows here
               
                    myXIRR.reset();
                    myXIRR.addCashflow( aScenario.First_Period_Start_Date__c, aScenario.First_Period_Cashflow__c );
                    myXIRR.addCashflow( aScenario.Second_Period_Start_Date__c, aScenario.Second_Period_Cashflow__c );
                    myXIRR.addCashflow( aScenario.Third_Period_Start_Date__c, aScenario.Third_Period_Cashflow__c );
                    
                    try {
                         // this method runs successive iterations to approximate the value of XIRR
                         aScenario.XIRR__c = myXIRR.calculate();
                         
                    } catch( XIRRHelper.XIRRException e ) {
                         aScenario.addError( e.getMessage() );
                    }
               }
          }
     }




