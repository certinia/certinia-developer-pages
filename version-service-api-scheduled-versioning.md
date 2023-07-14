---
permalink: /version-service-api-scheduled-versioning/
layout: page
---
# Version Service API - Scheduled Versioning

## Overview

Our Version Service API allows you to create versions of projects that contain a record of how a project has progressed. With project versions, you can compare previous and current details.

However, creating project versions isn't really something that comes to mind when you’re busy managing projects. Fortunately, we have good news: you can easily schedule a job to create project versions periodically using the Apex Scheduler and our API.

Sample Scenario: Project Versions Created Monthly

Using the Apex Scheduler you can schedule a job that runs once a month. Knowing this, you can have the job call a class with a method that calls our API and creates a project version.

## Implementation

1. Add a custom checkbox field to project named ‘Versioning__c’, which allows you to decide which projects get versioned each month and which don’t. We will use this field to query for Projects that we should create versions of.
2. Create global class that implements the Schedulable interface and have it call our Version Service API:

````
public class ScheduleVersioning implements Schedulable{
    public void execute(SchedulableContext ctx){
        //Call our versioning method 
        callVersioning();
    }

    private static void callVersioning() {
        //Query for Projects that need versioning
        List<pse__Proj__c> p = [SELECT Name, Id FROM pse__Proj__c WHERE Versioning__c = true];

        //Create list of versions
        List<pse.VersionService.Version> versions = new List<pse.VersionService.Version>();

        //Create versions for each project
        for(pse__Proj__c proj : p){
            Id projectId = proj.Id;
	 
            //Version setup
            pse.VersionService.Version dto = new pse.VersionService.Version();
            dto.ProjectId = projectId;
            dto.VersionName = 'Monthly Version ' + Date.Today();
            dto.Notes = 'Demo notes.';
            dto.Baseline = false;
	 
            versions.add(dto);
        }
        //Call API
        List<Id> versionIds = pse.VersionService.createAsync(versions);
    }
}
````

## Usage

Now all we have to do is call the Apex Scheduler to schedule our job.

Reminder: Make sure you have the correct permission controls set up for your selected projects to allow versioning.

This can be done in two ways:

### Execute Anonymous Apex

````
ScheduleVersioning sv = new ScheduleVersioning();
String cronExpr = '0 0 0 1 * ?';
System.schedule('Monthly Versioning', cronExpr, sv);
````

We use a CRON expression to state when we want our job to run. Here we have it set to run on the first of every month.

### Schedule from UI

1. From Setup, enter Apex in the Quick Find box, then select Apex Classes.
1. Click Schedule Apex.
1. For the job name, enter ‘Monthly Versioning’.
1. Click the lookup button next to Apex class and enter ‘ScheduleVersioning’ for the search term and click the name of your scheduled class.
1. Select ‘Monthly on day 1’ for the frequency, your preferred start and date dates, and ‘00:00’ for you preferred start time.
1. Click Save.

If you look at Scheduled Jobs in setup, you can see the job we scheduled and the next time it is set to run.

![All Scheduled Jobs](/assets/images/version-service-api-scheduled-versioning/001.jpeg)

With this implementation your projects are continually versioned even after being completed or set to inactive. You have to uncheck the 'Versioning' checkbox or set some sort of workflow validation rule that unchecks 'Versioning' when a project reaches completion or is set to inactive. 

## Notifications

It is already hard enough to remember your daily schedule, and you don’t need to add on a monthly task on top of that to check on. Good thing our API allows an easy way to send notifications and reminders.

1. Head over to Setup, enter ‘Custom Settings’ in the Quick Find box and select it.
1. Find and click Project Versioning Settings.
1. Click Manage.
1. Depending on if you want to set an org-wide default or a profile specific default click the top or bottom ‘New’ button respectively. For this example, we are selecting the top button for an org-wide default.
1. Enter your email address(es) in ‘Notifications_Recipients’ as a semicolon(;) separated list.
1. You can choose to receive notifications through chatter, email, or in the form of tasks. For this example, we select ‘Notifications Email’.
1. Click Save.

Now you’ll receive emails every month after your project versions have successfully been created.

Notifications also include job failures and version deletions.

## Unit Test

Simple enough right? Now all we have to do is test your Apex code:

````
@isTest
private class TestScheduleVersioning {
    @isTest static void testScheduleAndVersion() {
        //Create test data
        pse__Proj__c proj = new pse__Proj__c();
        proj.pse__Start_Date__c = Date.Today();
        proj.Name = 'Test Project';
        proj.pse__Is_Active__c = true;
        proj.Versioning__c = true;
        insert proj;

        //Permission controls to allow versioning
        pse__Permission_Control__c permissions = new pse__Permission_Control__c();
        permissions.pse__Compare_Project_Version__c = true;
        permissions.pse__Create_Project_Version__c = true;
        permissions.pse__Delete_Project_Version__c = true;
        permissions.pse__User__c = UserInfo.getUserId();
        permissions.pse__Project__c = proj.Id;
        insert permissions; 

        String cronExpr = '0 0 0 1 3 ? 2022';

        Test.StartTest();
        //Schedule the test job
        ScheduleVersioning sv = new ScheduleVersioning();
        System.schedule('test', cronExpr, sv);
        //Verify job has not run yet
        List<pse__Version__c> ver = [SELECT Name, Id FROM pse__Version__c WHERE pse__Project__c = 
            :proj.Id];
        System.assertEquals(0, ver.size(), 'Version created before job has run');
        //Job will run synchronously when stopping test
        Test.StopTest();
        //Verify version was created
        ver = [SELECT Name, Id FROM pse__Version__c WHERE pse__Project__c = :proj.Id];
        System.assertEquals(1, ver.size(), 'Version was not created');
    }
}
````
