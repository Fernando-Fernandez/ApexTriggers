# ApexTriggers
Different types of Apex triggers (Salesforce) and their helper classes

In all examples below the triggers will call static methods on the helper class and follow the format below:

```java
     trigger MyObjTrigger on MyObj__c ( before insert, before update, /* ... */ ) {
          MyObjHelperClass.myObjMethod( trigger.operationType, trigger.new, trigger.oldMap );
     }
```

All helper classes below will have a static method and a static set of ids to control recursion. The examples are intended to be as simple as possible and only to illustrate the concept below:

## Types of triggers
There are 6 most basic types of triggers (there may be others but these are the ones most common in my experience):

1. Trigger sets fields with values derived from other fields in the same record

2. Trigger adds validation error messages using data from fields in the same record

3. Trigger sets fields with values from other records

4. Trigger sets fields with values on other records

5. Trigger performs call outs to external web services or HTTP requests to receive data

6. Trigger sends an email

They are categorized according to where data comes from and goes to and their functional equivalent standard feature:

| Source of data | Destination of data | Trigger Type |
| ------------ | ------------ | ------------ |
| incoming records | incoming records valid status or error message | functional equivalent to a standard validation rule but implemented via trigger |
| incoming record fields | incoming record fields | functional equivalent to a standard formula field but implemented via trigger |
| other records | incoming records | functional equivalent to a standard cross-object formula field (or rollup) but implemented via trigger |
| incoming records | other records | trigger to propagate incoming records values to other records |
| incoming records and web service | incoming records | trigger to perform callouts to external web services, fetch data and store it |
| incoming records | outgoing email | functional equivalent to workflow alert |

------------
### 1 - Trigger that sets fields with values derived from other fields in the same record
This trigger does what a formula field would ideally do:  compute an expression using other field values. 
Some expressions are not possible using formula functions or the resulting expression exceeds the formula size limit, hence the requirement for this trigger. 
For example, there is no formula field that calculates the extended internal rate of return ([IRR or XIRR](https://en.wikipedia.org/wiki/Internal_rate_of_return)), which can only be calculated using Newton's iterative method.

```java
     trigger InvestmentTrigger on Investment__c ( before insert, before update ) {
          InvestmentHelperClass.calculateXIRR( trigger.operationType, trigger.new, trigger.oldMap );
     }
     
     class InvestmentHelperClass {
          public static void calculateXIRR( TriggerOperation operationType
                              , List<Investment__c> newList, Map<ID, Investment__c> oldMap ) {
               // TODO:  implement recursion prevention here
               
               XIRRHelper myXIRR = new XIRRHelper();
               
               for( Investment__c anInvestment : newList ) {
                    // TODO:  implement validation of periods and cashflows here
               
                    myXIRR.reset();
                    myXIRR.addCashflow( anInvestment.First_Period_Start_Date__c, anInvestment.First_Period_Cashflow__c );
                    myXIRR.addCashflow( anInvestment.Second_Period_Start_Date__c, anInvestment.Second_Period_Cashflow__c );
                    myXIRR.addCashflow( anInvestment.Third_Period_Start_Date__c, anInvestment.Third_Period_Cashflow__c );
                    
                    try {
                         // this method runs successive iterations to approximate the value of XIRR
                         anInvestment.XIRR__c = myXIRR.calculate();
                         
                    } catch( XIRRHelper.XIRRException e ) {
                         anInvestment.addError( e.getMessage() );
                    }
               }
          }
     }
```

The above trigger consists of a simple loop over the new/updated records. In the loop, it computes a value from some fields and stores the result in another field.

------------
### 2 - Trigger that adds validation error messages using data from fields in the same record
This trigger is similar to trigger #1 above, but it applies to validation. It does what a validation rule would ideally do:  check values from one or more fields against a criteria.
Some validation rules can't be expressed using validation formula functions or the resulting expression exceeds the validation size limit or require values from more than one record, hence the need for this trigger. 

```java
     trigger InvestmentTrigger on Investment__c ( before insert, before update ) {
          InvestmentHelperClass.validatePercentages( trigger.operationType, trigger.new, trigger.oldMap );
     }
     
     class InvestmentHelperClass {
          public static void validatePercentages( TriggerOperation operationType
                              , List<Investment__c> newList, Map<ID, Investment__c> oldMap ) {
               
               Map<Id, Decimal> percentageTotalsPerAccountIdMap = new Map<Id, Decimal>();
               
               for( Investment__c anInvestment : newList ) {
                    Decimal cumulativePercentage = percentageTotalsPerAccountIdMap.get( anInvestment.AccountId );
                    if( cumulativePercentage == null ) {
                         cumulativePercentage = 0;
                    }
                    cumulativePercentage = cumulativePercentage + anInvestment.Allocation_Percentage__c;
                    percentageTotalsPerAccountIdMap.put( anInvestment.AccountId, cumulativePercentage );

                    if( cumulativePercentage > 100.0 ) {
                         anInvestment.addError( 'Percentage of all investments under the same account must be at 100% or under' );
                    }
               }
          }
     }
```

Same as before:  the trigger consists of a simple loop over the new/updated records. This time, it determines the cumulative percentage of all investments under a same account.

------------
### 3 - Trigger that sets fields with values from other records
This trigger does what a cross object formula field would ideally do:  use values from other related records.
Sometimes the requirement goes beyond what a simple lookup/master-detail relationship could accommodate, hence the need for this trigger. 

The example below is adapted and simplified from a Field Service Lightning org. The client wanted a preferred resource automatically assigned to each work order. The resource needed to match the work order territory and be credentialed on the same service as the work order. 

```java
     trigger WorkOrderTrigger on WorkOrder ( before insert, before update ) {
          WorkOrderHelperClass.findPreferredResource( trigger.operationType, trigger.new, trigger.oldMap );
     }
     
     class WorkOrderHelperClass {
          public static void findPreferredResource( TriggerOperation operationType
                              , List<WorkOrder> newList, Map<ID, WorkOrder> oldMap ) {
               
               // collect territories and services to query ServiceTerritoryMember
               Set<String> workOrderTerritorySet = new Set<String>();
               Set<String> workOrderServiceSet = new Set<String>();
               for( WorkOrder aWorkOrder : newList ) {
                    if( aWorkOrder.ServiceTerritoryId != null ) {
                         workOrderTerritorySet.add( aWorkOrder.ServiceTerritoryId );
                    }
                    if( aWorkOrder.Service__c != null ) {
                         workOrderServiceSet.add( aWorkOrder.Service__c );
                    }
               }
               
               if( workOrderTerritorySet.isEmpty() && workOrderServiceSet.isEmpty() ) {
                    return;
               }
               
               // query ServiceTerritoryMember to build map indexed by territory and service
               List<ServiceTerritoryMember> resourceList = [
                    SELECT Id, ServiceResourceId, ServiceTerritoryId, ServiceResource.Service__c 
                    FROM ServiceTerritoryMember 
                    WHERE ServiceTerritoryId IN :workOrderTerritorySet 
                         AND ServiceResource.Service__c IN :workOrderServiceSet 
               ];
               
               Map<String, Id> resourceIdPerTerritoryServiceMap = new Map<String, Id>();
               for( ServiceTerritoryMember aResource : resourceList ) {
                    String indexKey = aResource.ServiceTerritoryId + '|' + aResource.ServiceResource.Service__c;
                    resourcesPerTerritoryServiceMap.put( indexKey, aResource );
               }
               
               // assign preferred resource matching by territory and service
               for( WorkOrder aWorkOrder : newList ) {
                    String indexKey = aWorkOrder.ServiceTerritoryId + '|' + aWorkOrder.Service__c;
                    Id aResourceId = resourceIdPerTerritoryServiceMap.put( indexKey );
                    aWorkOrder.Preferred_Resource__c = aResourceId;
               }
          }
     }
```

Notice that the pattern on the last 3 triggers was to start with a loop over the incoming records and use some data from them to take the next steps. In this case, the data was collected from the incoming records to perform a query on another object.

------------
### 4 - Trigger that sets fields with values on other records
This trigger simply propagates data from the incoming records to other objects in the org. This may be accomplished via record-triggered flows too but it tends to be more straightforward and performant to implement it in a trigger.

The example below is adapted from a Sales org client. When an opportunity was closed, the client wanted to create 12 months worth of child records. 
