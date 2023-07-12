---
permalink: /create-project-from-template-api-opportunity-product/
layout: page
---
# Create Project from Template API - Opportunity Product Automation

## Overview

Automating processes and functions can help a great deal in improving business performance and saving time. One process we can automate using our Create Project from Template Service is the creation of Projects for each Opportunity Product when an Opportunity becomes 'Closed Won'.

You can easily do this by creating an Apex trigger on the Opportunity object and adding a custom field to the Product object. The Apex trigger is what automatically calls the Create Project from Template Service and the lookup field from Product to Project allows specific templates to be used for an Opportunity Product depending on what Product is being offered instead of being restricted to using a one-size-fits-all template.

An Opportunity may have any number of Opportunity Products that link to template projects. So we could potentially be creating quite a few projects when the opportunity closes. By default, the CPFT API does not permit creating two projects simultaneously. If you need to create multiple projects at the same time, you can do this using queueables.

Warning: Asynchronous code has its own risks and rewards, especially when used in a trigger like we are here. See here. So be sure you understand what you are doing if you decide to use code like this.

## Implementation

1. Add a custom field to the Product object. Make it a lookup to the Project object and call it ‘Project_Template__c’.
2. Add a validation rule ‘Project_Template__r.pse__Is_Template__c = False’, so that a non-template can’t be linked in this field.
3. Create our class that calls the Create Project from Template Service:

````
public class CPFT {
    public static void createProject(Date startDate, Id templateProjectId, Id opportunityId, String projectName){
        
        // Instantiate the request.
        pse.CreateProjectFromTemplateService.CreateProjectFromTemplateAndOpportunityRequest 
            oppProjReq = new pse.CreateProjectFromTemplateService.CreateProjectFromTemplateAndOpportunityRequest(opportunityId, templateProjectId, startDate);
        
        oppProjReq.ProjectName = projectName;
 
        // Call the service to create the projects using the request list and store a list of responses.
        List<pse.CreateProjectFromTemplateService.CreateProjectResponse> oppProjResponse =
            pse.CreateProjectFromTemplateService.createProjectsFromTemplates( new 
            List<pse.CreateProjectFromTemplateService.CreateProjectFromTemplateAndOpportunityRequest>
            {oppProjReq});
        
        //Email error message if project not created successfully
        if(!oppProjResponse[0].isSuccess()){
            String eSubject = 'Error in creating ' + projectName;
            String eBody = 'Request for this project: ' + oppProjResponse[0].Request + 'nn' 
                + 'Errors related to the unsuccessful request: ' + oppProjResponse[0].Errors;
		
            sendEmail(eSubject, eBody);
        }
    }

    //Private method to send email
    private static void sendEmail(String subject, String body){
        Messaging.SingleEmailMessage message = new Messaging.SingleEmailMessage();
        String address = UserInfo.getUserEmail();
        message.toAddresses = new String[] { address };
        message.subject = subject;
        message.plainTextBody = body;
        Messaging.SingleEmailMessage[] messages = new List<Messaging.SingleEmailMessage> {message};
        Messaging.SendEmailResult[] results = Messaging.sendEmail(messages);
    }
}
````

4. Create a second class that implements the Queueable interface:

````
public with sharing class QueueCPFT implements Queueable{

    private Date startDate;
    private Id templateId;
    private Id oppId;
    private String projectName;

    public QueueCPFT(Date startDate, Id templateProjectId, Id oppId, String projectName){
        this.templateId = templateProjectId;
        this.startDate = startDate;
        this.oppId = oppId;
        this.projectName = projectName;
    }

    public void execute(QueueableContext context){
        CPFT.createProject(startDate, templateId, oppId, projectName);
    }
}
````

5. Create the Apex trigger is given below:

````
trigger CPFT_on_OPP_status on Opportunity (after update) {
    for( Opportunity opp : Trigger.new ){
        //Code will only run if the previous stage name was not 'Closed won'
        if(opp.StageName == 'Closed Won' && Trigger.oldMap.get(opp.Id).StageName != 'Closed Won'){

            Date startDate = opp.CloseDate;
            Id oppId = opp.Id;
	
            //OpportunityLineItem is the API name for Opportunity Product
            List<OpportunityLineItem> lineItems = new List<OpportunityLineItem>();
            //Query the Product2 name and Project lookup from Opportunity Line Items using OpportuntiyId 
            lineItems = [SELECT Name, Product2.Name, Product2.Project_Template__c FROM 
                OpportunityLineItem WHERE OpportunityId =:opp.Id];

            for(OpportunityLineItem oppProd : lineItems){
                if(oppProd.Product2.Project_Template__c != null){
	      String projectName = oppProd.Product2.Name + ' Project';
	      Id templateProjectId = oppProd.Product2.Project_Template__c;
	      QueueCPFT callCPFT = new QueueCPFT(startDate, templateProjectId, oppId, projectName);
	      //For test environment
	      //Chaining queueables not allowed in testing
	      if(Test.isRunningTest()){
	          callCPFT.execute(null);
	      }else{
	          System.enqueueJob(callCPFT);
	      }
                }
            }
        }
    }
}
````

## Usage

Now let’s see what all this code actually does. Take a look at our test Opportunity below:

![Opportunity](/assets/images/create-project-from-template-api-opportunity-product/001.jpg)

And look at all its Opportunity Products:

![Opportunity Products](/assets/images/create-project-from-template-api-opportunity-product/002.jpg)

Now mark the Opportunity as 'Closed won':

![Close This Opportunity](/assets/images/create-project-from-template-api-opportunity-product/003.jpg)

If you head over to back to view the test Opportunity, you can see that a Project was created for every Opportunity Product!

![Opportunity Projects](/assets/images/create-project-from-template-api-opportunity-product/004.jpg)

## Unit Test

Our Create Project from Template Service already calls a queueable method and chaining queueable methods is not allowed in a test environment which is why we have a ‘if(Test.isRunningTest())’ statement in our trigger. It’s always important to test our Apex code, so let’s look at some unit tests.

````
@isTest
private class TestCPFTTrigger {
	
    @isTest static void TestTrigger() {
        //Setup test data
        pse__Proj__c p = new pse__Proj__c();
        p.pse__Start_Date__c = Date.Today();
        p.Name = 'Test Project';
        p.pse__Is_Template__c = true;
        insert p;

        Id pricebookId = Test.getStandardPricebookId();
        	
        Opportunity o = new Opportunity();
        o.Name = 'Test Opportunity';
        o.CloseDate = Date.Today();
        o.StageName = 'Prospecting';
        insert o;

        Product2 p2 = new Product2();
        p2.Name = 'Test Product';
        p2.ProductCode = 'Test';
        p2.Project_Template__c = p.Id;
        insert p2;

        PricebookEntry pbe = new PricebookEntry();
        pbe.Pricebook2Id = pricebookId;
        pbe.Product2Id = p2.Id;
        pbe.UnitPrice = 100.00;
        pbe.IsActive = true;
        insert pbe;

        OpportunityLineItem oppLine = new OpportunityLineItem();
        oppLine.OpportunityId = o.Id;
        oppLine.PricebookEntryId = pbe.Id;
        oppLine.Quantity = 1;
        oppLine.TotalPrice = oppLine.Quantity*pbe.UnitPrice;
        insert oppLine;

        //Call method under test
        Test.StartTest();
        //Update stage name to call trigger
        o.StageName = 'Closed Won';
        update o;
        Test.StopTest();

         //Query to verify project was created and check opportunity name
        pse__Proj__c proj = [SELECT Name, Id, pse__Opportunity__r.Name FROM pse__Proj__c WHERE 
            Name = 'Test Product Project' LIMIT 1];
        System.assertEquals('Test Opportunity', proj.pse__Opportunity__r.Name);
    }
}
````