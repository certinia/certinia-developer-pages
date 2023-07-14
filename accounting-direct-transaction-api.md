---
permalink: /accounting-direct-transaction-api/
layout: page
---
# Direct Transaction API

## Overview

In previous versions of Certinia Accounting, posting would always require a source document, such as a sales invoice, and produce a transaction for that document. Users weren't able to have a transaction without a source document associated with it. There wasn't a direct route for creating transactions, which would cause some difficulties for integrations.

As of Accounting V15.0, we've delivered an API for the post process that will allow our customers, partners and our Product Services Team to create transactions directly without the need to create intermediate documents.

With the Open Transaction Model, transactions can be constructed directly, which may reduce data storage for some integrations and give them flexibility. If you provide values when creating transaction lines, these will not be overwritten by the posting process. You can do as much work as you need to in advance and the posting process will respect the information you provide.

All the same validations are carried out to enforce data consistency, and all the same currency conversions are also carried out. The post API will calculate document values in other currencies on the transaction, however for this version, default values are not automatically applied, such as the GLA of a product or an account, and no new lines are generated, such as tax lines. You must specify all the information for your transaction.

## AccountingTransaction and AccountingTransactionLine classes

A transaction in Accounting is composed of a transaction header and transaction lines. These objects correspond to the AccountingTransaction and AccountingTransactionLine classes of the API. So, an AccountingTransaction contains a list of AccountingTransactionLine objects. Both represent the FinancialForce Accounting transaction header and its lines.

- AccountingTransaction exposes a subset of fields of the Transaction object available for data input.
- AccountingTransactionLine exposes a subset of fields of the Transaction Line Item object available for data input.

Some of the remaining fields are calculated automatically by the system and others are internal to Accounting.

## API Syntax

````global static List post(List accountingTransactions, c2g.TransactionService.PostOptions postOptions)````

- AccountingTransaction is the object that must be completed to indicate the data that will be transferred to the Transaction.
- PostOptions configures special aspects of the post operation. In this release, it only supports the destination company where the transaction will be created.

To create a transaction you must create an instance of an AccountingTransaction and add to it as many AccountingTransactionLines as you need for your transaction. See the FinancialForce Accounting API Reference for details on the AccountingTransaction and AccountingTransactionLine fields.

For trade between EC countries, the sales invoice is often issued without VAT. When the recipient processes this as a payable invoice he must account for both, his input tax and the vendor's output tax, at the prevailing rate in his country of residence. Previously this would involve creating a journal from the payable invoice. Using the Direct PostAPI it is now possible to omit the journal and create the appropriate transaction lines directly.

## Sample Scenario: Reverse charge VAT

In the following example, we’re going to cover a similar scenario when a payable invoice is received from a vendor who is based in another EU country (e.g. the company is in Spain and the vendor is in Great Britain).

![Payable Invoice](/assets/images/accounting-direct-transaction-api/001.png)

Previously a journal had to be created to post the input and output tax, but now we’ll create the tax transaction directly.

Assume the payable invoice has a Tax Code of e.g. REV0 which has a lookup to REV20 with DR GLA = Output Reversal GLA and CR GLA = Input Reversal GLA then you'd end up with the following Journal:

| Line Number | Line Type | Tax Code | General Ledger Account | Value | Taxable Value |
| ----------- | --------- | -------- | ---------------------- | ----- | ------------- |
| 1 | Tax Code | REV20 | Output Reversal GLA | -17.88 | -89.40 |
| 2 | Tax Code | REV20 | Input Reversal GLA | 17.88 | 89.40 |

````
c2g.TransactionService.AccountingTransaction trans = new c2g.TransactionService.AccountingTransaction();
trans.setTransactionDate(trxDate);
trans.setPeriodId(periodId);
trans.setTransactionType(c2g.TransactionService.TransactionType.JOURNAL);
List<c2g.TransactionService.AccountingTransactionLine> transLines = new List<c2g.TransactionService.AccountingTransactionLine>();
// Tax line 1
{
    c2g.TransactionService.AccountingTransactionLine line = new c2g.TransactionService.AccountingTransactionLine();
    line.setLineType(c2g.TransactionService.TransactionLineType.TAX);
    line.setDocumentCurrencyId(docCurrencyId);
    line.setDueDate(trxDate);
    line.setTaxCode1Id(TaxCode1Id);
    line.setDocumentValue(-17.88);
    line.setGeneralLedgerAccountId(outputRevGLAId);
    transLines.add(line);
}
// Tax line 2
{
    c2g.TransactionService.AccountingTransactionLine line = new c2g.TransactionService.AccountingTransactionLine();
    line.setLineType(c2g.TransactionService.TransactionLineType.TAX);
    line.setDocumentCurrencyId(docCurrencyId);
    line.setDueDate(trxDate);
    line.setTaxCode1Id(TaxCode1Id);
    line.setDocumentValue(17.88);
    line.setGeneralLedgerAccountId(inputRevGLAId);
    transLines.add(line);
}
trans.TransactionLineItems = transLines;
List<ID> transIDs = c2g.TransactionService.post(new List<c2g.TransactionService.AccountingTransaction>{trans}, null);
````

## Sample Scenario: Post a transaction from a work order

In the following example, we’ve added a custom field to the transaction header and lines to make it clear that they have been created through the API.

![NewCustomFieldTransaction](/assets/images/accounting-direct-transaction-api/002.png)

In the following example, we’ve added a custom field to the transaction header and lines to make it clear that they have been created through the API.

![WorkOrderAndDetailObjects](/assets/images/accounting-direct-transaction-api/003.png)

````
Integer numOfLines=0, maxLines=15;
Decimal total = 0;
List<myCustomJournal__c> sourceLines = [Select GeneralLedgerAccount__c, Amount__c, Description__c, Currency__c, Period__c, Date__c From myCustomJournal__c];
if (sourceLines != null) 
   numOfLines=sourceLines.size();
List<c2g.TransactionService.AccountingTransaction> transactions = new List<c2g.TransactionService.AccountingTransaction>();
Integer i = 0;
Date trxDate;
Id docCurrency;
while (i < numOfLines)
{
    trxDate = sourceLines[i].Date__c;
    docCurrency = sourceLines[i].Currency__c;
    c2g.TransactionService.AccountingTransaction trans = new c2g.TransactionService.AccountingTransaction();
    trans.setTransactionDate(trxDate);
    trans.setPeriodId(sourceLines[i].Period__c);
    trans.setTransactionType(c2g.TransactionService.TransactionType.JOURNAL);
    List<c2g.TransactionService.AccountingTransactionLine> transLines = new List<c2g.TransactionService.AccountingTransactionLine>();
    c2g.TransactionService.AccountingTransactionLine line;
    Integer j = 1;
    while (j < maxLines && i < numOfLines)
    {
      // Analisys line
      line = new c2g.TransactionService.AccountingTransactionLine();
      line.setLineType(c2g.TransactionService.TransactionLineType.ANALYSIS);
      line.setDueDate(trxDate);
      line.setLineDescription(sourceLines[i].Description__c);
      // Document Information
      line.setDocumentCurrencyId(docCurrency);
      line.setDocumentValue(sourceLines[i].Amount__c);
      // GLA Information
      line.setGeneralLedgerAccountId(sourceLines[i].GeneralLedgerAccount__c);
      transLines.add(line);
      total += sourceLines[i].Amount__c;
      i++; 
      j++;
   }
   // Balance line
   line = new c2g.TransactionService.AccountingTransactionLine();
   line.setLineType(c2g.TransactionService.TransactionLineType.ANALYSIS);
   line.setDueDate(trxDate);
   // Document Information
   line.setDocumentCurrencyId(docCurrency);
   line.setDocumentValue(-1 * total);
   line.setGeneralLedgerAccountId(balanceGlaId);
   transLines.add(line);
   trans.TransactionLineItems = transLines;
   transactions.add(trans);
   total = 0;
}
List<ID> transIDs = c2g.TransactionService.post(transactions, null);
````

In this example, we assumed that all work orders belong to the same company and therefore will be posted in the current company. Also accountGLAId and productGLAId are pre-defined.

## Sample Scenario: Post a transaction on a specific company

The post API was designed to support multi-company so that an admin user (or a user with view all permission) will be able to define the company where transactions will be created without having to select it as current company.

In the following example, we are going to extend the previous scenario to allow the user to specify a destination company when creating transactions.

````
List<c2g.TransactionService.AccountingTransaction> transactions = new List<c2g.TransactionService.AccountingTransaction>();
for (WorkOrder__c workOrder : [Select id, Account__c, Currency__c, Date__c, Period__c, TotalAmount__c, (Select workOrderDetail__c.id, Hours__c, Rate__c, ServiceCode__c From Work_Order_Details__r) From WorkOrder__c)
{
    c2g.TransactionService.AccountingTransaction trans = new c2g.TransactionService.AccountingTransaction();
    trans.setTransactionDate(workOrder.Date__c);
    trans.setPeriodId(workOrder.Period__c);
    trans.setTransactionType(c2g.TransactionService.TransactionType.SALES_INVOICE);
    // Set custom field transaction without document
    trans.setFieldValue('TransactionWithoutDocument__c', true);
    // Add reference to source object via a custom lookup
    trans.setFieldValue('WorkOrder__c', workOrder.Id);
    List<c2g.TransactionService.AccountingTransactionLine> transLines = new List<c2g.TransactionService.AccountingTransactionLine>();
    // ACCOUNT
    c2g.TransactionService.AccountingTransactionLine line = new c2g.TransactionService.AccountingTransactionLine();
    line.setLineType(c2g.TransactionService.TransactionLineType.ACCOUNT);
    line.setAccountId(workOrder.Account__c);
    line.setGeneralLedgerAccountId(accountGLAId);
    line.setDueDate(workOrder.Date__c);
    // Document Information
    line.setDocumentCurrencyId(workOrder.Currency__c);
    line.setDocumentValue(workOrder.TotalAmount__c);
    // Set custom field transaction without document
    line.setFieldValue('TransactionWithoutDocument__c', true);
    transLines.add(line);
    List<WorkOrderDetail__c> workOrderLines = workOrder.Work_Order_Details__r;
    for (WorkOrderDetail__c sourceLine : workOrderLines)
    {
        // ANALYSIS
        line = new c2g.TransactionService.AccountingTransactionLine();
        line.setLineType(c2g.TransactionService.TransactionLineType.ANALYSIS);
        line.setProductId(sourceLine.ServiceCode__c);
        line.setGeneralLedgerAccountId(productGLAId);
        line.setDueDate(workOrder.Date__c);
        // Document Information
        line.setDocumentCurrencyId(workOrder.Currency__c);
        line.setDocumentValue(-1 * sourceLine.Hours__c * sourceLine.Rate__c);
        // Set custom field transaction without document
        line.setFieldValue('TransactionWithoutDocument__c', true);
        transLines.add(line);
    }
    trans.TransactionLineItems = transLines;
    transactions.add(trans);
}
List<ID> transIDs = c2g.TransactionService.post(transactions, null);
````

For this scenario, we assumed there are work orders that belong to different companies and therefore need to be posted into the corresponding company. Instead of creating a list of transactions to be posted directly, we created a map to group transactions by company. Then, we called the post method for each company, retrieving the list of transactions from the map and defining the destination company via the PostOptions. Only one destination company can be used for each post operation and if you want to create transactions for different companies, you have to perform different execution contexts for each company.

![BankStatementAndLinesObjects](/assets/images/accounting-direct-transaction-api/004.png)

Assuming that the source documents that will be paid are customer invoices and that the statement reference is the same of the invoice reference, we could run the following code to create cash entry transactions from a specific bank statement upload:

````
// Get the bank statement lines you want to process
List<c2g__codaBankStatementLineItem__c> statementLines = [Select id, c2g__Date__c , c2g__Amount__c, c2g__Reference__c From c2g__codaBankStatementLineItem__c Where c2g__BankStatement__c = :bankStatementId And c2g__BankStatement__r.c2g__BankAccount__c = :bankAccountId];
List<string> referenceList = new List<string>();
for (c2g__codaBankStatementLineItem__c line : statementLines)
{
  referenceList.add(line.c2g__Reference__c);
}
// Find the invoices referenced in the bank statement lines
List<c2g__codaInvoice__c> invoiceList = [Select id, c2g__Account__c, c2g__InvoiceTotal__c, c2g__InvoiceCurrency__c, c2g__CustomerReference__c, c2g__Account__r.c2g__codaAccountTradingCurrency__c From c2g__codaInvoice__c Where c2g__CustomerReference__c In :referenceList and c2g__InvoiceStatus__c = 'Complete' And c2g__OwnerCompany__c = :companyId];
Map<String, c2g__codaInvoice__c> invoiceByReference = new Map<String, c2g__codaInvoice__c>();
for (c2g__codaInvoice__c inv : invoiceList)
{
  invoiceByReference.put(inv.c2g__CustomerReference__c , inv);
}
// Create cash entry transactions from the statement lines
List<c2g.TransactionService.AccountingTransaction> transactions = new List<c2g.TransactionService.AccountingTransaction>();
for (c2g__codaBankStatementLineItem__c statement : statementLines)
{
  c2g__codaInvoice__c invoice = invoiceByReference.get(statement.c2g__Reference__c);
  if (invoice != null)
  {
    c2g.TransactionService.AccountingTransaction trans = new c2g.TransactionService.AccountingTransaction();
    trans.setTransactionDate(statement.c2g__Date__c);
    trans.setPeriodId(periodId);
    trans.setTransactionType(c2g.TransactionService.TransactionType.CASHENTRY);
    trans.setDocumentReference(invoice.c2g__CustomerReference__c);
    List<c2g.TransactionService.AccountingTransactionLine> transLines = new List<c2g.TransactionService.AccountingTransactionLine>();
    // ACCOUNT
    c2g.TransactionService.AccountingTransactionLine line = new c2g.TransactionService.AccountingTransactionLine();
    line.setLineType(c2g.TransactionService.TransactionLineType.ACCOUNT);
    line.setAccountId(invoice.c2g__Account__c);
    line.setGeneralLedgerAccountId(accountGLAId);
    line.setDueDate(statement.c2g__Date__c);
    // Document Information
    line.setDocumentCurrencyId(invoice.c2g__InvoiceCurrency__c);
    line.setDocumentValue(-1 * invoice.c2g__InvoiceTotal__c);
    // Override account currency if it's the same of the overridden currency (EUR)
    if (invoice.c2g__Account__r.c2g__codaAccountTradingCurrency__c == 'EUR')
      line.setAccountValue(-1 * statement.c2g__Amount__c);
    // Override home value if home currency is the same of the overridden currency (EUR)
    line.setHomeValue(-1 * statement.c2g__Amount__c);
    transLines.add(line);  
    // ANALYSIS - BANK ACCOUNT LINE
    line = new c2g.TransactionService.AccountingTransactionLine();
    line.setLineType(c2g.TransactionService.TransactionLineType.ANALYSIS);
    line.setBankAccountId(bankAccountId);
    line.setGeneralLedgerAccountId(bankAccountGLAId);
    line.setDueDate(statement.c2g__Date__c);
    // Document Information
    line.setDocumentCurrencyId(invoice.c2g__InvoiceCurrency__c);
    line.setDocumentValue(invoice.c2g__InvoiceTotal__c);
    // Override bank account value
    line.setBankAccountValue(statement.c2g__Amount__c);
    // Override bank account GLA value if GLA currency is the same of the overridden currency (EUR)
    line.setGeneralLedgerAccountValue(statement.c2g__Amount__c);
    // Override home value if home currency is the same of the overridden currency (EUR)
    line.setHomeValue(statement.c2g__Amount__c);
    transLines.add(line);
    trans.TransactionLineItems = transLines;            
    transactions.add(trans);        
  }
}
c2g.TransactionService.PostOptions postOptions = new c2g.TransactionService.PostOptions();
postOptions.DestinationCompanyId = companyId;
List<ID> transIDs = c2g.TransactionService.post(transactions, postOptions);
````

In the above example, we assumed that home currency and the bank account GLA currency are EUR, and the following referenced IDs are pre-defined: bankStatementId, bankAccountId, companyId, periodId, accountGLAId, bankAccountGLAId