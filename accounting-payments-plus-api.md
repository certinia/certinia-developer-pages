---
permalink: /accounting-payments-plus-api/
layout: page
---
# Payments Plus API

## Overview

Before Accounting V17, the only way to create and process a payment was via the Classic Payments user interface. Accounting V17 introduced a pilot version of Payments Plus as an improved way to handle payments. This new version, among other improvements, provides an API to create and process a payment without using the user interface. Since Accounting Spring 2018, Payments Plus is generally available and you can enjoy the benefits of this new API.

To explain in detail each method of this API, we have structured this article in two main sections. In the first one we describe, step by step, the Payments Plus process and show how each step corresponds to the global methods. In the second section, we provide a solution using the API for some sample use cases.

# Part 1: The Payments Plus Process

## Payments Plus Global Methods

The life cycle of a payment starts from the payment creation, until the payment is posted and matched to the ledgers and, in a later stage, it can also be canceled.

The steps needed to process a payment are listed in order:

1. Create a Payments Plus object.
1. Retrieve available transactions.
1. Add transactions to the payment proposal.
1. (Optional) Remove transactions from the payment proposal.
1. Create media tables.
1. (Optional) Remove accounts from the payment proposal (if the payment method is Electronic) or Update checks (if the payment method is Check).
1. Post and match the selected transactions.
1. (Optional) Discard those accounts that couldn’t be paid.
1. (Optional) Cancel a payment partially or totally.

We describe below each of the previous steps based on what users see when they use the application, and the API method(s) that corresponds to the different pages.

## Creating New Payment

The creation of a payment using Payments Plus can be seen in the following image. All the fields are mandatory, except the dimensions.

![001](/assets/images/accounting-payments-plus-api/001.png)

Let's create a new payment, using Check as the payment method:

````
c2g__codaPayment__c pay = new c2g__codaPayment__c();
        // Payment details
	pay.c2g__BankAccount__c = bankAccountId;
	pay.c2g__PaymentCurrency__c = paymentCurrencyId;
	pay.c2g__PaymentDate__c = system.today();
	pay.c2g__Period__c = periodId;
	pay.c2g__DiscountDate__c = system.today();
	pay.c2g__PaymentMediaTypes__c = 'Check';

	// Posting information
	pay.c2g__SettlementDiscountReceived__c = settlementDiscountReceivedId;
	pay.c2g__SDRDimension1__c = null;
	pay.c2g__SDRDimension2__c = null;
	pay.c2g__SDRDimension3__c = null;
	pay.c2g__SDRDimension4__c = null;

	pay.c2g__CurrencyWriteOff__c = currencyWriteOffId;
	pay.c2g__CWODimension1__c = null;
	pay.c2g__CWODimension2__c = null;
	pay.c2g__CWODimension3__c = null;
	pay.c2g__CWODimension4__c = null;
	pay.c2g__CreatedByPaymentsPlus__c = true;

	insert pay;
````

To simplify the code, we assumed that the referenced Ids (paymentCurrencyId, bankAccountId, periodId, settlementDiscountReceivedId and currencyWriteOffId) are predefined. For the c2g__PaymentMediaTypes__c, we only allow Check or Electronic. Also, note that the c2g__Status__c field is automatically set by the system.

## Retrieving Available Transactions

In the following image, you can click on Retrieve Transactions and a list of transactions are retrieved according to the filters.

![002](/assets/images/accounting-payments-plus-api/002.png)

You can create your own query to retrieve the transactions you want to pay and pass them to the "Add to Proposal" method. These queries can include custom fields. Also, you can perform as many queries as you want, to apply different filtering criteria, and add those transactions to your payment proposal, as long as:

1. The document currency matches the payment currency
1. The line type of the transaction line items is Account
1. The GLA is the same and matches the Accounts Payable Control on the account
1. And the retrieved transactions belong to your current company

## Adding Transactions to the Proposal

To add the retrieved transactions to the proposal, you can select the ones you want to pay and then click Add to Proposal. If there are errors, you’ll be notified.

This process can be repeated several times until you have added all the transactions you want to pay.

![003](/assets/images/accounting-payments-plus-api/003.png)

To perform this task via the API, just call:

````
c2g.PaymentsPlusService.addToProposal(pay.Id, transLineIdList);
````

where pay.Id corresponds to the Id of your payment (c2g__codaPayment__c) and transLineIdList is a list of Ids of the transaction line items (c2g__codaTransactionLineItem__c) you want to pay.

If there is a failure, this service returns a list containing all the new Payments Plus Error Log Ids (c2g__PaymentsPlusErrorLog__c) generated during the Add to Proposal process. If all transactions are added successfully, it will return an empty list.

## Removing Transactions from the Proposal

At this stage, if you decide you don’t want to pay some transactions, you still have the chance to remove them by clicking Remove from Proposal.

![004](/assets/images/accounting-payments-plus-api/004.png)

To perform this task via the API, just call:

````
c2g.PaymentsPlusService.removeFromProposal(pay.Id, transLineIdList);
````

where pay.Id corresponds to the Id of your payment (c2g__codaPayment__c) and transLineIdList is a list of Ids of the transaction line items (c2g__codaTransactionLineItem__c) you want to remove.

This service returns a list containing all the new Payments Plus Error Log Ids (c2g__PaymentsPlusErrorLog__c) generated during the Remove from Proposal process, in case of failure. If all transactions are added successfully, it will return an empty list.

## Creating Media Tables

Once you are satisfied with the payment proposal, the next step is to create the media tables. These tables are public and can be used by external tools to print checks or generate bank files for electronic transfer.

The following picture represents a payment proposal that will be paid using checks. Hence, once you click Prepare Checks, the media tables are generated in the background and a list of proposed check numbers are shown.

![005](/assets/images/accounting-payments-plus-api/005.png)

To perform this task via the API, you can call either the synchronous process:

````
c2g.PaymentsPlusService.createMediaData(pay.Id);
````

Or the asynchronous one:

````
c2g.PaymentsPlusService.createMediaDataAsync(pay.Id);
````

where pay.Id corresponds to the Id of your payment (c2g__codaPayment__c).

This service returns the ID of the Payment Media Control object related to the selected payment.

## Updating Checks

Once the media tables are created and a check proposal is prepared, you need to validate the check numbers before going to the next step, which will automatically update the media tables with the corresponding checks numbers and then post and match the payment. This is done once you click Post & Match.

![006](/assets/images/accounting-payments-plus-api/006.png)

The method for updating the media tables with the check numbers using the API is:

````
c2g.PaymentsPlusService.updateCheckNumbers(pay.Id, checks);
````

where pay.Id corresponds to the Id of your payment (c2g__codaPayment__c) and checks is a list of PaymentsPlusService.Check objects with the following properties:

1. AccountId: The ID for the account that you want to create a check for.
1. CheckNumber: The number you want to assign to the valid or void check.
1. Status: The status of the check you want to create or void. It can be StatusValid, StatusVoid or StatusManual.

This service returns the ID of the Payment Media Control object related to the selected payment.

## Discarding Payments Before Posting & Matching

If you’re running an Electronic payment, at this stage you can still remove transactions from the proposal by clicking Remove from Proposal. This option is allowed in case the bank has rejected some accounts. You have to provide the reason why you’re removing those transactions from the payment.

![007](/assets/images/accounting-payments-plus-api/007.png)

The corresponding API call is the following:

````
c2g.PaymentsPlusService.removeAccountsAsync(pay.Id, accountIdsList, removeAccountsReason);
````

where pay.Id corresponds to the Id of your payment (c2g__codaPayment__c), accountIdsList is a list of account Ids and removeAccountsReason is the reason why you are removing those accounts from the proposal. The removal reason is applied to all accounts in the list. If you wish to define different reasons for each account, you have to call this method again.

This service returns the ID of the batch process record for the job.

## Posting & Matching a Payment

Once all the changes are done and nothing else needs to be changed, you can post your payment to the ledgers. As we explained, this is done after clicking Post & Match.

![008](/assets/images/accounting-payments-plus-api/008.png)

The Post & Match can be invoked by calling the following method:

````
c2g.PaymentsPlusService.postAndMatchAsync(pay.Id);
````

where pay.Id corresponds to the Id of your payment (c2g__codaPayment__c).

This service returns the ID of the batch process record for the job or null if running off platform.

## Discarding Failed Payments After Post & Match

When the Post & Match process finishes with errors, the payment status is Error. At this stage, you can remove from the payment the transactions of those accounts that weren’t matched successfully. This will make them available so they can be paid and matched again. You can do this by just clicking Discard All failed Accounts.

![009](/assets/images/accounting-payments-plus-api/009.png)

You can do this via API by calling to the following method:

````
c2g.PaymentsPlusService.discardAsync(payment.Id);
````

where pay.Id corresponds to the Id of your payment (c2g__codaPayment__c).

This service returns the ID of the batch process record for the job.

## Canceling a Payment

Once the payment has been posted and matched, you have the option to cancel the whole payment or a subset of accounts. You might need to do this if, for example, a check has gone missing in the post.

If you want to cancel the whole payment, you should click Cancel -> All or, if you want to cancel some accounts, you should first select the accounts to be canceled and then click Cancel -> Selected.

![010](/assets/images/accounting-payments-plus-api/010.png)

Then you’ll have to provide a cancel reason and dates and periods for the undo match and payment refund cash entry.

![011](/assets/images/accounting-payments-plus-api/011.png)

If you want to cancel the whole payment, you have to call the following method:

````
c2g.PaymentsPlusService.cancelPaymentAsync(pay.Id, cancelPaymentCriteria);
````

Otherwise, for canceling a subset of accounts, you have to call:

````
c2g.PaymentsPlusService.cancelPaymentAsync(pay.Id, cancelPaymentCriteria, accountIDsList);
````

where pay.Id corresponds to the Id of your payment (c2g__codaPayment__c), accountIDsList is a list of account Ids and cancelPaymentCriteria is a c2g.PaymentsPlusService.CancelPaymentCriteria object with the following properties:

1. CancelReason: The reason why you want to cancel the accounts.
1. PaymentRefundDate: The date you want to use for the Refund Cash Entry.
1. PaymentRefundPeriod: The ID of the period you want to use for the Refund Cash Entry.
1. UndoMatchDate: The date you want to use for the underlying undo match operation.
1. UndoMatchPeriod: The ID of the period you want to use for the underlying undo match operation.

This service returns the ID of the batch process record for the job.

## Resubmitting the Cancel Payment

If the Cancel process fails, you can re-submit the cancel process for those accounts that couldn’t be canceled successfully, by clicking Resubmit Cancel Payment.

![012](/assets/images/accounting-payments-plus-api/012.png)

To do this using the API methods, call:

````
c2g.PaymentsPlusService.resubmitCancelPaymentAsync(pay.Id)
````

where pay.Id corresponds to the Id of the payment (c2g__codaPayment__c).

This service returns the ID of the batch process record for the job.

# Part 2: Payments Plus API Examples

## Sample Scenario 1: Pay invoices based on the outstanding amount

Suppose that I want to pay a large number of vendors based on a maximum invoice value, e.g. select all ‘available’ transactions line items (TLIs) where the document type is a payable invoice and the outstanding amount is up to £100:

````
// Build the list of codaTransactionLineItem__c that match our criteria, and add them to the proposal 
Set<Id> transLineIDs = new Map<Id,c2g__codaTransactionLineItem__c>([Select Id from c2g__codaTransactionLineItem__c where c2g__LineType__c = 'Account' and c2g__transaction__r.c2g__TransactionType__c = 'Invoice' and c2g__DocumentOutstandingValue__c >= -100 and c2g__DocumentOutstandingValue__c < 0 and c2g__MatchingStatus__c = 'Available']).keySet();
List<Id> transLineIDList = new List<Id> (transLineIDs);
c2g.PaymentsPlusService.addToProposal(pay.Id, transLineIDList);

// Once we have some transactions added to the payment proposal, we can add more transactions up to 10.000
// If the proposal is ready, the next step is create media data:
c2g.PaymentsPlusService.createMediaData(pay.Id);

// Because we chose a payment media based in checks, we need to add the checks information to the payment. 
// We must to assign a check for each account to be paid. So if our payment have just one account to paid (accountId) // and the first available check number in the active check range is 17:
List<c2g.PaymentsPlusService.Check> checks = new List<c2g.PaymentsPlusService.Check>();
c2g.PaymentsPlusService.Check check = new c2g.PaymentsPlusService.Check();
Id accountId;
check.AccountId = accountId;
check.CheckNumber = 17;
check.Status = c2g.PaymentsPlusService.CheckStatus.StatusValid;
checks.add(check);

// Once we've created the checks, we set them to the payment using: 
c2g.PaymentsPlusService.updateCheckNumbers(pay.Id, checks);

// Finally we can post and match the payment.
c2g.PaymentsPlusService.postAndMatchAsync(pay.Id);
````

## Sample Scenario 2: Pay invoices up to a maximum budget

Suppose I want to pay a large number of vendors based on the amount due to be paid, e.g. select all accounts with ‘available’ TLIs where document type is a payable invoice but this time the total outstanding amount for the account is no more than £1000. Once the payment is done, if the check is lost, we need to cancel the corresponding account:

````
// Build the list of codaTransactionLineItem__c that match our criteria, and add them to the proposal
List<nexusVerif__codaTransactionLineItem__c> transLineItems = [Select Id, nexusVerif__AccountOutstandingValue__c, NexusVerif__Account__c from nexusVerif__codaTransactionLineItem__c where nexusVerif__LineType__c = 'Account' and nexusVerif__transaction__r.nexusVerif__TransactionType__c = 'Purchase Invoice' and nexusVerif__MatchingStatus__c = 'Available'];

List<nexusVerif__codaTransactionLineItem__c> transLineItems = [Select Id, nexusVerif__AccountOutstandingValue__c, NexusVerif__Account__c from nexusVerif__codaTransactionLineItem__c where nexusVerif__LineType__c = 'Account' and nexusVerif__transaction__r.nexusVerif__TransactionType__c = 'Purchase Invoice' and nexusVerif__MatchingStatus__c = 'Available'];

// List of account Ids to be paid

Map<Id, List<Id>> tlisByAccountId = new Map<Id, List<Id>>();
Map<Id, Decimal> totalAmountByAccountId = new Map<Id, Decimal>();

Id accountId;
for (nexusVerif__codaTransactionLineItem__c tli: transLineItems)
{
	accountId = tli.NexusVerif__Account__c;
	if (totalAmmountByAccountId.get(accountId)==null)
	{
   		 tlisByAccountId.put(accountId, new List<Id>());
   		 totalAmountByAccountId.put(accountId, 0);
	}
	Decimal totalAmount = totalAmountByAccountId.get(accountId) + tli.nexusVerif__AccountOutstandingValue__c;
	if (totalAmount*-1<1000)
	{
  	tlisByAccountId.get(accountId).add(tli.Id);
  	totalAmmountByAccountId.put(accountId, totalAmount);
	}
}

List<Id> transLineIDList = new List<Id>();
for (List<Id> tlis : tlisByAccountId.Values())
{
	transLineIDList.addAll(tlis);
}
c2g.PaymentsPlusService.addToProposal(pay.Id, transLineIDList);

// We proceed with the same process than the previous case: create media data and update check numbers
c2g.PaymentsPlusService.createMediaData(pay.Id);

List<c2g.PaymentsPlusService.Check> checks = new List<c2g.PaymentsPlusService.Check>();
c2g.PaymentsPlusService.Check check = new c2g.PaymentsPlusService.Check();
Id accountId;
check.AccountId = accountId;
check.CheckNumber = 1;
check.Status = c2g.PaymentsPlusService.CheckStatus.StatusValid;
checks.add(check);
c2g.PaymentsPlusService.updateCheckNumbers(pay.Id, checks);

// Post and match
c2g.PaymentsPlusService.postAndMatchAsync(pay.Id);

// In case that you want to cancel an account:
list<ID> accountIds = new list<ID>();
accountIds.add(ID(s) of the account(s) to be canceled);
c2g.PaymentsPlusService.CancelPaymentCriteria cancelPaymentCriteria = new c2g.PaymentsPlusService.CancelPaymentCriteria();
cancelPaymentCriteria.CancelReason = 'Cancel Reason';
cancelPaymentCriteria.PaymentRefundDate = system.today();
cancelPaymentCriteria.PaymentRefundPeriod = periodId;
cancelPaymentCriteria.UndoMatchDate = system.today();
cancelPaymentCriteria.UndoMatchPeriod = periodId;
c2g.PaymentsPlusService.cancelPaymentAsync(pay.Id, cancelCriteria, accountIds);
````

## Sample Scenario 3: Void checks

Suppose I want to pay all invoices on my Accounting Payable ledger that are due for payment up to 31st March 2018 and set the check numbers to start at 200651 instead of 200650, with 200650 being marked as void:

````
// Build the list of codaTransactionLineItem__c that match our criteria, and add them to the proposal
List<nexusVerif__codaTransactionLineItem__c> transLineItems = [Select Id, nexusVerif__AccountOutstandingValue__c, NexusVerif__Account__c from nexusVerif__codaTransactionLineItem__c where nexusVerif__LineType__c = 'Account' and nexusVerif__transaction__r.nexusVerif__TransactionType__c = 'Purchase Invoice' and nexusVerif__MatchingStatus__c = 'Available'];

List<nexusVerif__codaTransactionLineItem__c> transLineItems = [Select Id, nexusVerif__AccountOutstandingValue__c, NexusVerif__Account__c from nexusVerif__codaTransactionLineItem__c where nexusVerif__LineType__c = 'Account' and nexusVerif__transaction__r.nexusVerif__TransactionType__c = 'Purchase Invoice' and nexusVerif__MatchingStatus__c = 'Available'];

// List of account Ids to be paid

Map<Id, List<Id>> tlisByAccountId = new Map<Id, List<Id>>();
Map<Id, Decimal> totalAmountByAccountId = new Map<Id, Decimal>();

Id accountId;
for (nexusVerif__codaTransactionLineItem__c tli: transLineItems)
{
	accountId = tli.NexusVerif__Account__c;
	if (totalAmmountByAccountId.get(accountId)==null)
	{
   		 tlisByAccountId.put(accountId, new List<Id>());
   		 totalAmountByAccountId.put(accountId, 0);
	}
	Decimal totalAmount = totalAmountByAccountId.get(accountId) + tli.nexusVerif__AccountOutstandingValue__c;
	if (totalAmount*-1<1000)
	{
  	tlisByAccountId.get(accountId).add(tli.Id);
  	totalAmmountByAccountId.put(accountId, totalAmount);
	}
}

List<Id> transLineIDList = new List<Id>();
for (List<Id> tlis : tlisByAccountId.Values())
{
	transLineIDList.addAll(tlis);
}
c2g.PaymentsPlusService.addToProposal(pay.Id, transLineIDList);

// We proceed with the same process than the previous case: create media data and update check numbers
c2g.PaymentsPlusService.createMediaData(pay.Id);

List<c2g.PaymentsPlusService.Check> checks = new List<c2g.PaymentsPlusService.Check>();
c2g.PaymentsPlusService.Check check = new c2g.PaymentsPlusService.Check();
Id accountId;
check.AccountId = accountId;
check.CheckNumber = 1;
check.Status = c2g.PaymentsPlusService.CheckStatus.StatusValid;
checks.add(check);
c2g.PaymentsPlusService.updateCheckNumbers(pay.Id, checks);

// Post and match
c2g.PaymentsPlusService.postAndMatchAsync(pay.Id);

// In case that you want to cancel an account:
list<ID> accountIds = new list<ID>();
accountIds.add(ID(s) of the account(s) to be canceled);
c2g.PaymentsPlusService.CancelPaymentCriteria cancelPaymentCriteria = new c2g.PaymentsPlusService.CancelPaymentCriteria();
cancelPaymentCriteria.CancelReason = 'Cancel Reason';
cancelPaymentCriteria.PaymentRefundDate = system.today();
cancelPaymentCriteria.PaymentRefundPeriod = periodId;
cancelPaymentCriteria.UndoMatchDate = system.today();
cancelPaymentCriteria.UndoMatchPeriod = periodId;
c2g.PaymentsPlusService.cancelPaymentAsync(pay.Id, cancelCriteria, accountIds);
````

## Sample Scenario 4: Pay invoices from a specific region

Suppose that I want to pay all invoices that are due for payment on the Expenses Ledger for the Northern Region, e.g. select all 'available' TLIs where document type is a payable invoice and the Accounting Payable Control GLA of the account is 'Expenses' and Dimension 1 = 'Northern':

````
// Build the list of codaTransactionLineItem__c that match our criteria, and add them to the proposal
Id accountPayableControlGlaId;
Id northernDimId;
Set<Id> transLineIDs = new Map<Id,c2g__codaTransactionLineItem__c>([Select Id from c2g__codaTransactionLineItem__c where c2g__LineType__c = 'Account' and c2g__transaction__r.c2g__TransactionType__c = 'Purchase Invoice' and c2g__GeneralLedgerAccount__c =: accountPayableControlGlaId and c2g__Dimension1__c =: northernDimId and c2g__MatchingStatus__c = 'Available']).keySet();
List<Id> transLineIDList = new List<Id> (transLineIDs);
c2g.PaymentsPlusService.addToProposal(pay.Id, transLineIDList);

// Create the media data. Payment media is electronic
c2g.PaymentsPlusService.createMediaData(pay.Id);

// Post and match
c2g.PaymentsPlusService.postAndMatchAsync(pay.Id);
````

## Sample Scenario 5: Pay invoices approved through a Purchase Order

Suppose that I want to pay all invoices that have been approved by matching a purchase order, e.g. select all 'available' TLIs where the document type is payable invoice and Purchase Order Approved (custom field) is True:

````
// Build the list of codaTransactionLineItem__c that match our criteria and add them to the proposal. 
// We assume that c2g__codaTransactionLineItem__c has the custom field purchaseOrderApproved__c
Set<Id> transLineIDs = new Map<Id,c2g__codaTransactionLineItem__c>([Select Id from c2g__codaTransactionLineItem__c where c2g__LineType__c = 'Account' and c2g__transaction__r.c2g__TransactionType__c = 'Invoice' and purchaseOrderApproved__c = true and c2g__MatchingStatus__c = 'Available']).keySet();
List<Id> transLineIDList = new List<Id> (transLineIDs);
c2g.PaymentsPlusService.addToProposal(pay.Id, transLineIDList);

// Create the media data. Payment media is electronic
c2g.PaymentsPlusService.createMediaData(pay.Id);

// Post and Match
c2g.PaymentsPlusService.postAndMatchAsync(pay.Id);
````
