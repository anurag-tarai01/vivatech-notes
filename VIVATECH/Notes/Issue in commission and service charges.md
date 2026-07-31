The issue was with grade mapping for `SUBSCRIBER_CASH_IN`.

For cash-in transactions:

- Sender = Agent
    
- Receiver = Subscriber
    

But in `setServiceChargeAndCommission()`, the code was always passing:

- `subscriberGradeId` as `senderGradeId`
    
- `agentGradeId` as `receiverGradeId`
    

This worked for `SUBSCRIBER_CASH_OUT`, but not for `SUBSCRIBER_CASH_IN`, because for cash-in the actual sender is the agent.

Additionally, `agentGradeId` and `payeeCategories` were never populated for `SUBSCRIBER_CASH_IN`, since none of the existing agent-detection conditions matched that transfer type.

Because of this, the service charge and commission queries were executed with incorrect inputs:

- `senderGradeId = Subscriber`
    
- `receiverGradeId = empty`
    
- `payeeCategories = []`
    

So both service charge and commission slab lookups returned `size=0`.

Fix implemented:

1. Added dedicated handling for `SUBSCRIBER_CASH_IN` to fetch sender agent info from `fromAccountAggregateId`.
    
2. Passed grades in correct sender/receiver order while creating `ServiceChargeProfileCalculateRequestDto`.
    

After the fix:

- Service charge slab lookup is returning records correctly.
    
- Commission slab lookup is also working correctly for `SUBSCRIBER_CASH_IN`.