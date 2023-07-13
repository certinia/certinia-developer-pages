---
permalink: /create-project-from-template-api-global-action-button/
layout: page
---
# Create Project from Template API - Global Action Button

## Overview

Sometimes your Project needs to be created quickly and efficiently. With the Create Project from Template Service API, you can create a project from a template with the click of a button without requiring a full-featured, sophisticated UI. With this API, you can do this easily in a few steps on any and as many object layouts as you desire.

Sample Scenario: Global Action Button

You can add a default button to any object layout that enables users to create projects using a template of your choosing. Consider your company has multiple cases for services or products that must be fulfilled into projects. You can speed this up by creating a button that easily creates projects by calling our Create Project from Template Service.

## Implementation

1\. Add a lookup field from Case to Project named ‘Project_Template__c’. Use the lookup filter: 'Project Template: Template EQUALS True'.

2\. Create an Apex class that we will use as a controller with this code:

````
public class SimpleCPFTController {

    //Used to get input fields for Project Lookup and Name
    public Case myCase{get;set;}
    
    //Used to get input fields for date
    public pse__Proj__c inputProj{
        get{
            if(inputProj == null)
                inputProj = new pse__Proj__c();
            return inputProj;
        }set;}

    public SimpleCPFTController(ApexPages.StandardController stdController) {
        myCase = (Case)stdController.getRecord();
     }

    public PageReference createProject(){
        //Set up input fields for template service
        Id templateProjectId = myCase.Project_Template__c; 
        Date startDate = inputProj.pse__Start_Date__c;

        //Instantiate the request
        pse.CreateProjectFromTemplateService.CreateProjectFromTemplateRequest projRequest =
            new pse.CreateProjectFromTemplateService.CreateProjectFromTemplateRequest(templateProjectId, 
            startDate);

        //Set new project name to specified name
        projRequest.ProjectName = myCase.Subject;

        //Call the service and create project
        List<pse.CreateProjectFromTemplateService.CreateProjectResponse> projResponse =
            pse.CreateProjectFromTemplateService.createProjectsFromTemplates( 
            new List<pse.CreateProjectFromTemplateService.CreateProjectFromTemplateRequest>{projRequest});

        //If project was created successfully then redirect
        if(projResponse[0].isSuccess()){
            //Obtain new project id and redirect to project page
            Id ProjectId = projResponse[0].NewProjectId;
            PageReference nextPage = new PageReference('/' + ProjectId);
            nextPage.setRedirect(true);
            return nextPage;
        //If unsuccessful output error messages
        }else{
            for(pse.CreateProjectFromTemplateService.CreateProjectError err : projResponse[0].Errors){
                ApexPages.addMessage(new ApexPages.message(ApexPages.severity.ERROR, err.Message));
            }
            return null;
        }
    }
}
````

3\. Create a Visualforce page to call the Apex controller created above. Use “CreateProjectForCase” for the name

````
<apex:page standardController="Case" extensions="SimpleCPFTController">
    <apex:pageBlock title="Create Project From Template">
        <apex:pageMessages />
        <apex:form >
            <apex:pageBlockSection columns="1">
                
                <apex:inputField value="{!Case.Subject}" label="Project Name" />
                <apex:inputField value="{!Case.Project_Template__c}" label="Project Template" />
                <apex:inputField value="{!inputProj.pse__Start_Date__c}" label="Start Date" />
                <apex:commandButton action="{!createProject}" value="Create Project From Template" />
                             
            </apex:pageBlockSection>
        </apex:form>
    </apex:pageBlock>
</apex:page>
````
4\. Create a Detail Page button to call our Visualforce page. Make sure the Content Source is 'Visualforce Page' and the Content is the 'CreateProjectForCase' page we just made.

![Create a Detail Page button](/assets/images/create-project-from-template-api-global-action-button/001.jpg)

5\. Add the Detail Page button to the Case object page layout.

![Add the Detail Page button to the Case object page layout](/assets/images/create-project-from-template-api-global-action-button/002.jpg)

## Usage

![Detail Page with button](/assets/images/create-project-from-template-api-global-action-button/003.jpg)

As you can see in this screenshot, we now have a Create Project From Template button in the top right. In this scenario, the button redirects to a custom Visualforce page.

![Create Project From Template Form](/assets/images/create-project-from-template-api-global-action-button/004.jpg)

Now let’s fill out the input fields, and create a project!

![Create Project From Template Form - date](/assets/images/create-project-from-template-api-global-action-button/005.jpg)

After clicking Create Project From Template, you’re redirected to your newly created project.

![New Project record](/assets/images/create-project-from-template-api-global-action-button/006.jpg)

![New Project Resource Requests](/assets/images/create-project-from-template-api-global-action-button/007.jpg)

As you can see, the project has been created with all the details present in our template along with any objects such as Resource Requests, Milestones, Project Tasks, and etc.

Instead of just throwing a bunch of code at you, let’s talk about what we’ve done and how it all works.

### Visualforce:

1. Created a Visualforce page that uses a controller extension we wrote, as well as using the standard Case controller.
2. Added three input fields, a ‘Project Name’ field for the name of the project which we are going to create, a lookup field for Projects which is filtered to only be able to search up templates, and a third field which is for the start date of our project.
3. Add a clickable button that calls our Apex method that takes our input fields, creates the project, then redirects to the created project.

### Apex:

We use an Apex controller extension on the Visualforce page. It has the following features:

1. We retrieve the case record when the page is loaded.
2. It has two properties to bind data between the three input fields on the Visualforce page and the apex code: ‘myCase’ and ‘inputProj’.
3. It has an Apex method, ‘createProject’. When you press the button on the Visualforce page it calls this method, which in turn calls the Create Project from Template Service to create our project and then redirects you to the newly created project page.

Finally, all Apex code deserves some unit tests which is required on the Salesforce platform.

````
@isTest
private class TestSimpleCPFTController {
    
    @isTest static void testButton() {
        Case c = new Case();
        c.Subject = 'Test Case Project';
        c.Status = 'Escalated';
        c.Origin = 'Phone';

        pse__Proj__c p = new pse__Proj__c();
        p.pse__Start_Date__c = Date.Today();
        p.Name = 'Test';
        p.pse__Is_Template__c = true;
        insert p;
        c.Project_Template__c = p.Id;
        insert c;

        Test.StartTest();
        ApexPages.StandardController sc = new ApexPages.StandardController(c);
        SimpleCPFTController scc = new SimpleCPFTController(sc);
        scc.inputProj = p;
        PageReference returnPage = scc.createProject();
        Test.StopTest();
    
        //Check input fields from Visualforce page match
        System.assertEquals(p.Id, scc.myCase.Project_Template__c, 'Project template does not match');
        System.assertEquals('Test Case Project', scc.myCase.Subject, 'Inputted Project name does not match');
        System.assertEquals(Date.Today(), scc.inputProj.pse__Start_Date__c, 'inputted start date does not match');
        pse__Proj__c proj = [SELECT Id, Name FROM pse__Proj__c WHERE Name='Test Case Project' LIMIT 1];
        PageReference testPage = new PageReference('/' + proj.Id);
        //Check we are redirected correctly
        System.assertEquals(testPage.getURL(), returnPage.getURL(), 'Not redirected to correct page');
    }
    
    @isTest static void testGetProject(){
        Case c = new Case();
        ApexPages.StandardController sc = new ApexPages.StandardController(c);
        SimpleCPFTController scc = new SimpleCPFTController(sc);
        Test.StartTest();        
        pse__Proj__c proj = scc.inputProj;
        Test.StopTest();
        
        System.assertNotEquals(null, proj, 'dummy project should be created by controller');
    }

    @isTest static void testNoStartDate(){
        Case c = new Case();
        c.Subject = 'Test Case Project';
        c.Status = 'Escalated';
        c.Origin = 'Phone';

        pse__Proj__c p = new pse__Proj__c();
        p.Name = 'Test';
        p.pse__Is_Template__c = true;
        insert p;
        c.Project_Template__c = p.Id;
        insert c;

        Test.StartTest();
        ApexPages.StandardController sc = new ApexPages.StandardController(c);
        SimpleCPFTController scc = new SimpleCPFTController(sc);
        scc.inputProj = p;
        PageReference returnPage = scc.createProject();
        Test.StopTest();

        List<Apexpages.Message> msg = ApexPages.getMessages();
        String errMsg = 'The request must provide a start date.';
        System.debug(msg[0].getDetail());
        System.assertEquals(msg[0].getDetail(), errMsg);
    }
}
````

## Follow-on Scenario: Linking back to the case

Now that we’ve created a project from the Case detail page, other team members can get on with the work in that project to get the case resolved. You can further enhance the workflow by adding a custom field to the Project object linking to the related Case. Clicking on the custom field would take you to the related case from the project.

Our above example didn’t set the Case on the Project. There is nothing to stop you doing it manually once the project is created. You could even update the code to run some extra DML and update the project automatically. What would be even better is if we could set the Case field on the Project as it’s created. The CPFT API lets you do just that.

## Implementation

1. Add a lookup field from Project to Case called Case__c.
2. Replace the code in SimpleCPFTController with the following:

````
public class SimpleCPFTController {

    //Used to get input fields for Project Lookup and Name
    public Case myCase{get;set;}
    
    //Used to get input fields for date
    public pse__Proj__c inputProj{
        get{
            if(inputProj == null)
                inputProj = new pse__Proj__c();
            return inputProj;
        }set;}

    public SimpleCPFTController(ApexPages.StandardController stdController) {
        myCase = (Case)stdController.getRecord();
     }

    public PageReference createProject(){
        //Instantiate the request
        pse.CreateProjectFromTemplateService.CreateProjectFromTemplateRequest projRequest = createRequest();

        //Call the service and create project
        List<pse.CreateProjectFromTemplateService.CreateProjectResponse> projResponse =
            pse.CreateProjectFromTemplateService.createProjectsFromTemplates( 
            new List<pse.CreateProjectFromTemplateService.CreateProjectFromTemplateRequest>{projRequest});

        //If project was created successfully then redirect
        if(projResponse[0].isSuccess()){
            //Obtain new project id and redirect to project page
            Id ProjectId = projResponse[0].NewProjectId;
            PageReference nextPage = new PageReference('/' + ProjectId);
            nextPage.setRedirect(true);
            return nextPage;
        //If unsuccessful output error messages
        }else{
        
        System.debug(LoggingLevel.ERROR, projResponse);
            for(pse.CreateProjectFromTemplateService.CreateProjectError err : projResponse[0].Errors){
            
            System.debug(LoggingLevel.ERROR, 'Adding Error: ' + err.Message);
                ApexPages.addMessage(new ApexPages.message(ApexPages.severity.ERROR, err.Message));
            }
            return null;
        }
    }
    
    private pse.CreateProjectFromTemplateService.CreateProjectFromTemplateRequest createRequest()
    {
        //Set up input fields for template service
        Id templateProjectId = myCase.Project_Template__c;
        Date startDate = inputProj.pse__Start_Date__c;
        
        //Instantiate the request
        pse.CreateProjectFromTemplateService.CreateProjectFromTemplateRequest projRequest =
            new pse.CreateProjectFromTemplateService.CreateProjectFromTemplateRequest(templateProjectId, 
            startDate);

        //Set new project name to specified name
        projRequest.ProjectName = myCase.Subject;
        
        //Make a fieldMapper for the Case field on Project. Set the default value to the case we are using.
        pse.SObjectCloneMapper.Field caseMapper = new pse.SObjectCloneMapper.Field(Schema.pse__Proj__c.Case__c);
        caseMapper.DefaultValue = myCase.Id;
            
        Set<pse.SObjectCloneMapper.Field> setOfFields = new Set<pse.SObjectCloneMapper.Field>{caseMapper};

        //Make a mapper for the Project object
        pse.SObjectCloneMapper itemCloneMap = new pse.SObjectCloneMapper(pse__Proj__c.SObjectType, setOfFields);
            
        projRequest.Mappers = new List<pse.SObjectCloneMapper> { itemCloneMap };

        return projRequest;
    }
}
````

Once we’ve added the field to Project, we just need to change the code a little bit and the CPFT API can start populating it. Let’s look at the changes we made in this code.

All the changes are in the Request object that we pass into the API. (We’ve also extracted the code that makes the request into a new method to make it easier to follow.) The request is how we tell the API everything it needs to know to make our projects 'just right'. Here we added a SObjectCloneMapper.Field for the Case field on Project. And told it to set it to the Id of the Case we are working with. We then need to put that ‘FieldMapper’ in a set and link it to the Project object, and we are done.

## Unit tests

We just need up update the one 'happy path' test to check we are setting the Case field.

````
@isTest static void testButton() {
        Case c = new Case();
        c.Subject = 'Test Case Project';
        c.Status = 'Escalated';
        c.Origin = 'Phone';

        pse__Proj__c p = new pse__Proj__c();
        p.pse__Start_Date__c = Date.Today();
        p.Name = 'Test';
        p.pse__Is_Template__c = true;
        insert p;
        c.Project_Template__c = p.Id;
        insert c;

        Test.StartTest();
        ApexPages.StandardController sc = new ApexPages.StandardController(c);
        SimpleCPFTController scc = new SimpleCPFTController(sc);
        scc.inputProj = p;
        PageReference returnPage = scc.createProject();
        Test.StopTest();
    
        //Check input fields from Visualforce page match
        System.assertEquals(p.Id, scc.myCase.Project_Template__c, 'Project template does not match');
        System.assertEquals('Test Case Project', scc.myCase.Subject, 'Inputted Project name does not match');
        System.assertEquals(Date.Today(), scc.inputProj.pse__Start_Date__c, 'inputted start date does not match');
        pse__Proj__c proj = [SELECT Id, Name, Case__c FROM pse__Proj__c WHERE Name='Test Case Project' LIMIT 1];
        System.assertEquals(c.Id, proj.Case__c, 'Case field on project does not match');
        
        PageReference testPage = new PageReference('/' + proj.Id);
        //Check we are redirected correctly
        System.assertEquals(testPage.getURL(), returnPage.getURL(), 'Not redirected to correct page');
    }
````