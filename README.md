# ApexTriggers
Different types of Apex triggers (Salesforce) and their helper classes

## SUMMARY:  these are 6 types of common triggers that cover estimated 70% of Apex development needs (*).

In all examples below the triggers will call static methods on the helper class and follow the format below:

```java
trigger MyObjTrigger on MyObj__c ( before insert, before update, /* ... */ ) {
     MyObjHelperClass.myObjMethod( trigger.operationType, trigger.new, trigger.oldMap );
}
```

All helper classes below are implied to have static methods and set of ids to control recursion. The examples are intended to be as simple as possible and only to illustrate the concepts below:

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

          if( resourceList.isEmpty() ) {
               return;
          }

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

Notice that the pattern on the last 3 triggers was to start with a loop over the incoming records. Then it leverages data from them to take the next steps. In this case, the data was collected from the incoming records to perform a query on another object. This leads to the common trigger pattern loop-query-loop-update.

Another pattern is to build a map right after a query. The map is usually indexed not by Id but by one field or a combination of fields. Then the map is used as an in-memory lookup inside a loop.

------------
### 4 - Trigger that sets fields with values on other records
This trigger simply propagates data from the incoming records to other objects in the org, usually to children records or to parent records. This may be accomplished via record-triggered flows too but it tends to be more straightforward and performant to implement it in a trigger.

The example below is adapted from a Sales org client. When an opportunity was closed, the client wanted to create 12 months worth of child records (Disbursement__c). 
Since the child records need the parent opportunity id, this trigger is after insert/update.

```java
trigger OpportunityTrigger on Opportunity ( after insert, after update ) {
     OpportunityHelperClass.createDisbursements( trigger.operationType, trigger.new, trigger.oldMap );
}

class OpportunityHelperClass {
     public static void createDisbursements( TriggerOperation operationType
                         , List<Opportunity> newList, Map<ID, Opportunity> oldMap ) {

          // create Disbursements for opportunities that became closed won
          List<Disbursement__c> newDisbursementList = new List<Disbursement__c>();
          for( Opportunity anOpportunity : newList ) {
               // skip if opportunity was not changed to closed won (or inserted as closed won)
               Opportunity oldOpportunity = oldMap?.get( anOpportunity.Id );
               if( anOpportunity.IsClosed == oldOpportunity?.IsClosed 
                   && anOpportunity.IsWon == oldOpportunity?.IsWon ) {
                    continue;
               }

               // create template Disbursement linked to opportunity
               Disbursement__c templateDisbursement = new Disbursement__c();
               aDisbursement.Opportunity__c = anOpportunity.Id;

               // create Disbursements for each month cloned from template
               for( Integer i = 0; i < 12; i++ ) {
                   Disbursement__c newDisbursement = templateDisbursement.clone();
                   newDisbursement.Disbursement_Number__c = i + 1;
                   newDisbursement.Name = anOpportunity.Name + ' - Disbursement ' + newDisbursement.Disbursement_Number__c;
                   newDisbursement.Disbursement_Date__c = anOpportunity.CloseDate.addMonths( i );

                   newDisbursementList.add( newDisbursement );
               }
           }

           if( ! newDisbursementList.isEmpty() ) {
               insert newDisbursementList;
           }
     }
}
```

The example below is adapted from another Sales org client. When an opportunity product changed its status, the client wanted it to automatically sync the parent opportunity status.

```java
trigger OpportunityLineItemTrigger on OpportunityLineItem ( after insert, after update ) {
     OpportunityLineItemHelperClass.syncParentOpportunityStatus( trigger.operationType, trigger.new, trigger.oldMap );
}

class OpportunityLineItemHelperClass {
     public static void syncParentOpportunityStatus( TriggerOperation operationType
                         , List<OpportunityLineItem> newList, Map<ID, OpportunityLineItem> oldMap ) {

           // collect parent opportunity ids
           Set<Id> opportunityIdSet = new Set<Id>();
           for( OpportunityLineItem anOpportunityLineItem : newList ) {
               // skip if OpportunityLineItem status was not changed
               OpportunityLineItem oldOpportunityLineItem = oldMap?.get( anOpportunityLineItem.Id );
               if( anOpportunityLineItem.Status__c == oldOpportunityLineItem?.Status__c ) {
                   continue;
               }

               opportunityIdSet.add( anOpportunityLineItem.OpportunityId );
           }

           if( opportunityIdSet.isEmpty() ) {
                return;
           }

           // fetch parent opportunities with their opportunity products
           List<Opportunity> opportunityList = [
               SELECT Id, StageName
                   , ( SELECT Id, Status__c 
                       FROM OpportunityLineItems )
               FROM Opportunity
               WHERE Id IN : opportunityIdSet
           ];

           if( opportunityList.isEmpty() ) {
               return;
           }

           // update opportunities if applicable
           List<Opportunity> opportunitiesForUpdateList = new List<Opportunity>();
           for( Opportunity anOpportunity : opportunityList ) {
               if( anOpportunity.OpportunityLineItems == null
                       || anOpportunity.OpportunityLineItems.isEmpty() ) {
                   continue;
               }

               String currentStage = anOpportunity.StageName;
               Boolean inProgress = false;
               for( OpportunityLineItem anOpportunityLineItem : anOpportunity.OpportunityLineItems ) {
                   // if a new product was added to a won opportunity, set it to Reopened Won
                   if( anOpportunityLineItem.Status__c == 'New' 
                           && currentStage == 'Closed Won' ) {
                       currentStage == 'Reopened Won';
                       continue;
                   }

                   if( anOpportunityLineItem.Status__c == 'In Progress' ) {
                       inProgress = true;
                       continue;
                   }
               }

               // check if at least one line item is in progress
               if( inProgress ) {
                   currentStage = 'In Progress Won';
               }

               // only update if stage needs to be synced
               if( currentStage != anOpportunity.StageName ) {
                   anOpportunity.StageName == currentStage;
                   opportunitiesForUpdateList.add( anOpportunity );
               }
           }

           if( ! opportunitiesForUpdateList.isEmpty() ) {
               update opportunitiesForUpdateList;
           }
     }
}
```

These triggers almost always start with an initial loop over the incoming records but here we observe another pattern:  it collects data from the incoming records, then use that data to query some other object, then loop over the queried data to perform some update. This loop-query-loop-update pattern is very common.

------------
### 5 - Trigger that performs call outs to external web services or HTTP requests to receive data

This trigger collects data from the incoming records, calls out a web service passing that data as parameters, then updates one or more objects in the org. This requires the use of @future(callout=true) since 

The example below was adapted from a client org that needed the transportation distance and duration between one of their warehouses and their customers whenever the warehouse or the customer address was changed on the record. We leveraged the Distance Matrix API from Google.

```java
trigger OpportunityTrigger on Opportunity ( after insert, after update ) {
     OpportunityHelperClass.calculateTransportationCosts( trigger.operationType, trigger.new, trigger.oldMap );
}

class OpportunityLineItemHelperClass {
     public static void calculateTransportationCosts( TriggerOperation operationType
                         , List<Opportunity> newList, Map<ID, Opportunity> oldMap ) {

           // collect opportunity source-destination pairs
           Map<Id, String> opportunitySourceMap = new Map<Id, String>();
           Map<Id, String> opportunityDestinationMap = new Map<Id, String>();
           for( Opportunity anOpportunity : newList ) {
               // skip if Warehouse__c and AccountId were not changed
               Opportunity oldOpportunity = oldMap?.get( anOpportunity.Id );
               if( anOpportunity.Warehouse__c == oldOpportunity?.Warehouse__c
                       && anOpportunity.AccountId == oldOpportunity?.AccountId ) {
                   continue;
               }

               opportunitySourceMap.put( anOpportunity.Id, anOpportunity.Warehouse_Address__c );
               opportunityDestinationMap.put( anOpportunity.Id, anOpportunity.Customer_Address__c );
           }

           calculateDistance( opportunitySourceMap, opportunityDestinationMap );
     }

     @future( callout=true )
     public static void calculateDistance( Map<Id, String> opportunitySourceMap
                                       , Map<Id, String> opportunityDestinationMap ) {

           // get distance matrix from Google API
           GoogleAPIHelper.DistanceMatrix distanceMatrix = GoogleAPIHelper.getDistanceMatrix( 
                                               opportunitySourceMap, opportunityDestinationMap );

           List<Opportunity> opportunityList = [
               SELECT Id, Warehouse_Address__c, Customer_Address__c 
               FROM Opportunity 
               WHERE Id IN :opportunitySourceMap.keySet() 
           ];

           // calculate transportation costs
           for( Opportunity anOpportunity : opportunityList ) {
               anOpportunity.Transportation_Cost__c = distanceMatrix.getCost( 
                                           opportunitySourceMap.get( anOpportunity.Id ), 
                                           opportunityDestinationMap.get( anOpportunity.Id ) );
           }

           update opportunityList;
     }
}
```

This one was a depature from the pattern:  instead of loop-query-loop-update, it was a loop-async callout plus query-loop-update.

------------
### 6 - Trigger that sends emails related to incoming records

This trigger collects data from the incoming records, then sends an email linked to the record according to a criteria. Ideally this would be handled with a workflow alert or a record-triggered flow, but some criteria may be too complex to be implemented satisfactorily in a flow/rule.

In the example below, the client wanted to be notified when a quote belonging to a parent opportunity was changed to no longer be the opportunity's only quote flagged as forecast.

```java
     trigger QuoteTrigger on Quote ( after insert, after update ) {
          QuoteHelperClass.checkForecastQuoteChange( trigger.operationType, trigger.new, trigger.oldMap );
     }
     
     class QuoteLineItemHelperClass {
          public static void checkForecastQuoteChange( TriggerOperation operationType
                              , List<Quote> newList, Map<ID, Quote> oldMap ) {
               
                // collect parent opportunity ids to retrieve them
                Set<Id> opportunityIdSet = new Set<Id>();
                Map<Id, String> QuoteDestinationMap = new Map<Id, String>();
                for( Quote aQuote : newList ) {
                    // skip if Forecast__c was not changed
                    Quote oldQuote = oldMap?.get( aQuote.Id );
                    if( aQuote.Forecast__c == oldQuote?.Forecast__c ) {
                        continue;
                    }

                    opportunityIdSet.add( aQuote.OpportunityId );
                }

                // fetch parent opportunities with their quotes
                List<Opportunity> opportunityList = [
                    SELECT Id, Type
                        , ( SELECT Id, Forecast__c 
                            FROM Quotes )
                    FROM Opportunity
                    WHERE Id IN : opportunityIdSet
                ];

                if( opportunityList.isEmpty() ) {
                    return;
                }

                // send emails if applicable
                List<Messaging.SingleEmailMessage> emailList = new List<Messaging.SingleEmailMessage>();
                for( Opportunity anOpportunity : opportunityList ) {
                    if( anOpportunity.Quotes == null || anOpportunity.Quotes.isEmpty() ) {
                        continue;
                    }

                    Integer forecastCount = 0;
                    for( Quote aQuote : anOpportunity.Quotes ) {
                         if( aQuote.Forecast__c == true ) {
                            forecastCount ++;
                         }

                         if( forecastCount > 1 ) {
                             Messaging.SingleEmailMessage message = getForecastQuoteChangedMessage( 
                                                                     anOpportunity.Id );
                             emailList.add( message );

                             // skip rest of the quotes for this opportunity
                             break;
                         }
                    }
                }

                if( emailList.isEmpty() ) {
                    return;
                }

                Messaging.SendEmailResult[] results = Messaging.sendEmail( emailList );

                // TODO: handle results
          }
     }
```

Another variation of the loop-query-loop-update:  loop-query-loop-email.

------------
(*) this is taken from personal experience working on hundreds of orgs over a period of 8+ years.
