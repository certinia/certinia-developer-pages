---
permalink: /version-service-api-milestone-status/
layout: page
---
# Version Service API - Versioning on Milestone Status

## Overview

Our Version Service API allows you to create versions of Projects. It’s important to have Project versions. It helps maintain a record of how a Project has progressed and allows you to look back on previous details or past versions of a Project. However, creating project versions isn't really something that comes to mind when you’re busy managing projects. For example, let’s say you want to know what a project looked like when it moved from phase 1 to phase 2. Fortunately, there is good news: we can use the Process Builder and an invocable method to automatically call our API and create a version of the Project.

Sample Scenario: Project Versions on Approved Milestones

When a Milestone is approved, it usually signifies an important or big step for a Project. This marks a good point for when Project versions should be created. You can do this by using the Process Builder on Milestone whenever its status is changed to 'Approved'.
‘’
## Implementation

1\. Create an Apex class with a invocable method that calls our Version Service API:

````
global class InvocableVersioning {

    @InvocableMethod(label='Create Project Version ' description='Create Project Version')
    global static void createProjectVersion (List<pse__Milestone__c> milestone) {
        List<pse.VersionService.Version> versions = new List<pse.VersionService.Version>();
	 
        //Version setup.
        pse.VersionService.Version dto = new pse.VersionService.Version();
        dto.ProjectId = milestone[0].pse__Project__c;
        dto.VersionName = 'Approved ' + milestone[0].Name + ' Version ' + Date.Today();
        dto.Notes = 'Version automatically created on approved Milestone: ' + milestone[0].Name;
        dto.Baseline = false;
	 
        versions.add(dto);
        //Call API
        List<Id> versionIds = pse.VersionService.createAsync(versions);
     }
}
````

Along with the date in the version name, the notes tell us which milestone triggered the version to be created.

2\. Create a new Process with the Process Builder and select 'A record changes' for 'The process starts when'.

![New Process](/assets/images/version-service-api-milestone-status/001.jpeg)

3\. Add the ‘Milestone’ object we are working on and select ‘When a record is created or edited’.

![Milestone](/assets/images/version-service-api-milestone-status/002.jpeg)

4\. Add criteria for our following action. Select ‘Conditions are met’ and ‘All of the conditions are met’. For the Set Conditions, add:

- Status Equals Picklist Approved
- Status Is changed Boolean True

![Conditions](/assets/images/version-service-api-milestone-status/003.jpeg)

Note: you could change the implementation to also create versions when approved milestones are inserted.

5\. Finally, create an action and set the type to ‘Apex’ and the class to our invocable method name. For the Apex Variable, we select our ‘milestone’ field and set it to our Milestone object

![Apes Action](/assets/images/version-service-api-milestone-status/004.jpeg)

6\. Now let’s activate our process and see it in action.

![Activate](/assets/images/version-service-api-milestone-status/005.jpeg)

If an error occurs from versioning, the process builder states that the flow failed to trigger instead of the Version Service error message. Alternative means to communicate errors (update a field on the record, set a status, required permission controls, etc…) should be explored if the standard platform error handling in process builder is not acceptable to your users.

## Usage

Next, If we go to our project page we can see its milestones:

Reminder: make sure the user has the correct permission controls set up to allow versioning.

![Project Milestones](/assets/images/version-service-api-milestone-status/006.jpeg)

Let’s change ‘Milestone1’ to ‘Approved’

![Edit Milestone1](/assets/images/version-service-api-milestone-status/007.jpeg)

Now if we look at ‘Managed Versions’ on our Project page, we can see our newly created Project version.

![Project related](/assets/images/version-service-api-milestone-status/008.jpeg)

Clicking it takes us to the version detail page.

![Version detail](/assets/images/version-service-api-milestone-status/009.jpeg)

And in the related tab are all the Version’s related objects.

![Version related](/assets/images/version-service-api-milestone-status/010.jpeg)

## Unit Test

Now let's look at a unit tests for our code:

````
@isTest
private class TestInvocableVersioning {
    @isTest static void TestUpdate() {
        //Set up test data
        pse__Proj__c proj = new pse__Proj__c();
        proj.pse__Start_Date__c = Date.Today();
        proj.Name = 'Test Project';
        proj.pse__Is_Active__c = true;
        insert proj;

        //Set up permission controls to allow versioning
        pse__Permission_Control__c permissions = new pse__Permission_Control__c();
        permissions.pse__Compare_Project_Version__c = true;
        permissions.pse__Create_Project_Version__c = true;
        permissions.pse__Delete_Project_Version__c = true;
        permissions.pse__User__c = UserInfo.getUserId();
        permissions.pse__Project__c = proj.Id;
        insert permissions; 

        //More test data
        pse__Milestone__c m = new pse__Milestone__c();
        m.Name = 'Test Milestone';
        m.pse__Target_Date__c = Date.Today();
        m.pse__Project__c = proj.Id;
        m.pse__Milestone_Amount__c = 4;
        m.pse__Status__c = 'Planned';
        insert m;
        
        List<pse__Milestone__c> milestones = new List<pse__Milestone__c> ();
        milestones.add(m);

        Test.startTest();
        //Call invocable method
        InvocableVersioning.createProjectVersion(milestones);
        Test.stopTest();

        //Query for created version
        List<pse__Version__c> ver = [SELECT Name, pse__Notes__c FROM pse__Version__c WHERE pse__Project__c = :proj.Id];
		
        //See if version was created
        System.assertEquals(1, ver.size(), 'Version was not created');
        String versionName = 'Approved Test Milestone Version ' + Date.Today();
        //See if name matches
        System.assertEquals(versionName, ver[0].Name);
        System.assertEquals('Version automatically created on approved Milestone: Test Milestone', ver[0].pse__Notes__c);
    }
}
````
