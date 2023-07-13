---
permalink: /create-project-from-template-api-custom-objects/
layout: page
---
# Create Project from Template API - Working with Custom Objects

## Overview

When creating a project from a template, you can copy your own custom objects. You can copy over any arrangement of custom objects related to a template using the 'SObjectCloneMapper' and the 'Mappers' property from the Create Project from Template Service.

The SObjectCloneMapper defines the structure of SObjects that you want to clone. The Create Project from  Template Service automatically uses the SObjectCloneMapper to copy over project template data in the following objects by default:

- Resource Request
- Milestone
- Assignment
- Assignment Project Phase
- Assignment Milestone
- Assignment Project Methodology
- Project Methodology
- Project Task Assignment
- Project Phase
- Project Task
- Project Task Dependency
- Project Location
- Schedule
- Schedule Exception

Sample Scenario: Copying corresponding Checklist Items for Project Phases

Consider that your company has a custom object named 'Checklist Items' which has a lookup to the Project Phase object. Checklist Items have a checkbox field named 'Done'. A phase is designated as finished when all Checklist Items are marked as done. You’ve added specific Phases and Checklist Items to a template and you want to include the Checklist Items when you call the Create Project from Template Service to create your new Project.

The following screenshot shows some of an example template and its Project Phases.

![an example template and its Project Phases](/assets/images/create-project-from-template-api-custom-objects/001.jpg)

Clicking one of the Project Phases takes you to its detail page and here are its Checklist Items in the related tab.

![Checklist Items in the related tab](/assets/images/create-project-from-template-api-custom-objects/002.jpg)

## Implementation

The following code calls the Create Project from Template Service which copies over the fields and data from the above template along with the Checklist Items into a new Project. This code can be built into a trigger, an invokable method, or any of the other customizations available on the Salesforce platform, allowing you to create a Project complete with Checklist Items without changing any of your company’s business processes.

````
public with sharing class CPFTIncludingCheckListItems {
    public static void callCPFTIncludingCheckListItems(Id templateProjectId, Date startDate, String projectName) {
        
        //Clone mapper fields for checklist item
        //Lookup field required in order for related objects to be copied over
        pse.SObjectCloneMapper.Field itemMapperField1 = new 
            pse.SObjectCloneMapper.Field(Schema.Checklist_Item__c.Project_Phase__c);
        pse.SObjectCloneMapper.Field itemMapperField2 = new 
            pse.SObjectCloneMapper.Field(Schema.Checklist_Item__c.Name);
        pse.SObjectCloneMapper.Field itemMapperField3 = new 
            pse.SObjectCloneMapper.Field(Schema.Checklist_Item__c.Done__c);
        //Checklist items are set to unchecked as default
        itemMapperField3.DefaultValue = false;

        //Group together the pse.SObjectCloneMapper.Field instantiated above for each object and prepare a set.
        Set<pse.SObjectCloneMapper.Field> setItemMapperFields =
            new Set<pse.SObjectCloneMapper.Field>{itemMapperField1, itemMapperField2, itemMapperField3};
 
        //Instantiate the SObjectCloneMapper using the prepared set of SObjectCloneMapper.Field for each object.
        pse.SObjectCloneMapper itemCloneMap = new 
            pse.SObjectCloneMapper(Checklist_Item__c.SObjectType, setItemMapperFields);

        //Instantiate the request
        pse.CreateProjectFromTemplateService.CreateProjectFromTemplateRequest projRequest = 
            new pse.CreateProjectFromTemplateService.CreateProjectFromTemplateRequest(templateProjectId, 
            startDate);

        projRequest.ProjectName = projectName;
        projRequest.Mappers = new List<pse.SObjectCloneMapper> { itemCloneMap };
        
        //Call the service to create the projects using the request list and store a list of responses.
        List<pse.CreateProjectFromTemplateService.CreateProjectResponse> projResponses = 
            pse.CreateProjectFromTemplateService.createProjectsFromTemplates( new 
            List<pse.CreateProjectFromTemplateService.CreateProjectFromTemplateRequest>{projRequest});
    }
}
````

And of course, all Apex code deserve some unit tests:

````
@isTest
private class TestCPFTIncludingCheckListItems {
	
    @isTest static void testChecklistItems() {
        pse__Proj__c p = new pse__Proj__c();
        p.pse__Start_Date__c = Date.Today();
        p.Name = 'Test';
        p.pse__Is_Template__c = true;
        insert p;
		
        pse__Project_Phase__c phase = new pse__Project_Phase__c();
        phase.Name = 'Phase Test';
        phase.pse__Project__c = p.Id;
        insert phase;

        Checklist_Item__c ci = new Checklist_Item__c();
        ci.Name = 'Checklist Item Test';
        ci.Done__c = true;
        ci.Project_Phase__c = phase.Id;
        insert ci;

        Date d = Date.Today();
        String s = 'Test Case Project';

        Test.StartTest();
        CPFTIncludingCheckListItems.callCPFTIncludingCheckListItems(p.Id, d, s);
        Test.StopTest();
		
        Checklist_Item__c checkItem = [SELECT Name, Done__c, Project_Phase__r.Name, 
            Project_Phase__r.pse__Project__r.Name, 
            Project_phase__r.pse__Project__r.pse__Start_Date__c FROM Checklist_Item__c 
            WHERE Project_Phase__r.pse__Project__r.Name ='Test Case Project' LIMIT 1];
        
        //Check Checklist Item fields and successfully created Project and related fields
        System.assertEquals(false, checkItem.Done__c);
        System.assertEquals(‘Checklist Item Test’, checkItem.Name);
        System.assertEquals('Phase Test',checkItem.Project_Phase__r.Name);
        System.assertEquals(s, checkItem.Project_Phase__r.pse__Project__r.Name);
        System.assertEquals(d, checkItem.Project_phase__r.pse__Project__r.pse__Start_Date__c);
    }
}
````

## Other Possibilities

We only showed a simple example of what’s possible with the SObject Cloner. As you know, you can copy over any arrangement of custom objects and related objects from a template using the 'SObjectCloneMapper' and the 'Mappers' property of the Create Project from Template Service. To show more of what this API is capable of doing, let’s add on to our previous example with a new custom object, ‘Materials’. The Materials object refers to any materials you may need for your project. Materials also contains lookup fields to another custom object ‘Material Group’ which groups materials together. Material also has a look up to 'Company' and detail fields 'Quantity' and 'Unit of Measurement'.

You want to copy over Materials and its fields along with the Checklist Items you’re already copying over. Fortunately, the SObjectCloneMapper allows you to do just that. Although Material Group itself doesn’t look up to project, Material does, and the API can work out which Material Groups to copy for you. On the other hand, Material also looks up to company, and we don’t want to copy companies every time we copy a project. The API can do that too, it copies the material and sets the Company lookup to reference the same Company as the template material. All this is set up in the request you make to the API.

The following screenshot shows a template and its Materials

![a template and its Materials](/assets/images/create-project-from-template-api-custom-objects/003.jpg)

The Material Page is shown below

![The Material Page](/assets/images/create-project-from-template-api-custom-objects/004.jpg)

## Implementation

The following code copies the Material object along with its fields and look up to the Material Group and Company objects. The Material Group is copied over as well along with Checklist Item from the template into your new project.

````
public with sharing class CPFTIncludingChecklistItemsAndMaterials {
    public static void callCPFTIncludingChecklistItemsAndMaterials(Id templateProjectId, Date startDate, String projectName) {
        
        //Clone mapper fields for checklist item
        //Lookup field required in order for related objects to be copied over
        pse.SObjectCloneMapper.Field itemMapperField1 = new 
            pse.SObjectCloneMapper.Field(Schema.Checklist_Item__c.Project_Phase__c);
        pse.SObjectCloneMapper.Field itemMapperField2 = new 
            pse.SObjectCloneMapper.Field(Schema.Checklist_Item__c.Name);
        pse.SObjectCloneMapper.Field itemMapperField3 = new 
            pse.SObjectCloneMapper.Field(Schema.Checklist_Item__c.Done__c);
        //Checklist items are set to unchecked as default
        itemMapperField3.DefaultValue = false;

        //Clone mapper fields for Material
        //Lookup field required in order for related objects to be copied over
        pse.SObjectCloneMapper.Field materialMapperField1 = new 
            pse.SObjectCloneMapper.Field(Schema.Material__c.Name);
        pse.SObjectCloneMapper.Field materialMapperField2 = new 
            pse.SObjectCloneMapper.Field(Schema.Material__c.Project__c);
        pse.SObjectCloneMapper.Field materialMapperField3 = new 
            pse.SObjectCloneMapper.Field(Schema.Material__c.Quantity__c);
        pse.SObjectCloneMapper.Field materialMapperField4 = new 
            pse.SObjectCloneMapper.Field(Schema.Material__c.Unit_of_Measurement__c);
        pse.SObjectCloneMapper.Field materialMapperField5 = new 
            pse.SObjectCloneMapper.Field(Schema.Material__c.Company__c);
        pse.SObjectCloneMapper.Field materialMapperField6 = new 
            pse.SObjectCloneMapper.Field(Schema.Material__c.Material_Group__c);

        //Clone mapper fields for Material Group
        pse.SObjectCloneMapper.Field groupMapperfield1 = new
            pse.SObjectCloneMapper.Field(Schema.Material_Group__c.Name);
        
        //Group together the pse.SObjectCloneMapper.Field instantiated above for each object and prepare a set.
        Set<pse.SObjectCloneMapper.Field> setItemMapperFields =
            new Set<pse.SObjectCloneMapper.Field>{itemMapperField1, itemMapperField2, itemMapperField3};

        Set<pse.SObjectCloneMapper.Field> setMaterialMapperFields =
            new Set<pse.SObjectCloneMapper.Field>{materialMapperField1, materialMapperField2, 
            materialMapperField3, materialMapperField4, materialMapperField5, materialMapperField6};

        Set<pse.SObjectCloneMapper.Field> setGroupMapperFields = 
            new Set<pse.SObjectCLoneMapper.Field>{groupMapperfield1};
 
        //Instantiate the SObjectCloneMapper using the prepared set of SObjectCloneMapper.Field for each object.
        pse.SObjectCloneMapper itemCloneMap = new 
            pse.SObjectCloneMapper(Checklist_Item__c.SObjectType, setItemMapperFields);
        pse.SObjectCloneMapper materialCloneMap = 
            new pse.SObjectCloneMapper(Material__c.SObjectType, setMaterialMapperFields);
        pse.SObjectCloneMapper groupCloneMap = 
            new pse.SObjectCloneMapper(Material_Group__c.SObjectType, setGroupMapperFields);


        //Instantiate the request
        pse.CreateProjectFromTemplateService.CreateProjectFromTemplateRequest projRequest =
            new pse.CreateProjectFromTemplateService.CreateProjectFromTemplateRequest(templateProjectId, 
            startDate);

        projRequest.ProjectName = projectName;
        projRequest.Mappers = new List<pse.SObjectCloneMapper> 
            { itemCloneMap, materialCloneMap, groupCloneMap };
        
        //Call the service to create the projects using the request list and store a list of responses.
        List<pse.CreateProjectFromTemplateService.CreateProjectResponse> projResponses = 
            pse.CreateProjectFromTemplateService.createProjectsFromTemplates( new 
            List<pse.CreateProjectFromTemplateService.CreateProjectFromTemplateRequest>{projRequest});
    }
}
````

So, what’s going on here? Why do Material Groups get copied but not Companies? That’s because we provided a Mapper for Material Group, but not for Company. If we had added a Mapper for Company too, then the CPFT Service would have copied companies for us. On the other hand, if we hadn’t provided the Material Group Mapper, then the CPFT wouldn’t copy Material Groups and the materials in the cloned project would look up to the same groups as the materials in the template.

And of course, the unit test:

````
@isTest
private class testCPFTInclChecklistItemsAndMaterials {
    @isTest static void test_method_one() {
        pse__Proj__c p = new pse__Proj__c();
        p.pse__Start_Date__c = Date.Today();
        p.Name = 'Test';
        p.pse__Is_Template__c = true;
        insert p;
		
        pse__Project_Phase__c phase = new pse__Project_Phase__c();
        phase.Name = 'Phase Test';
        phase.pse__Project__c = p.Id;
        insert phase;

        Checklist_Item__c ci = new Checklist_Item__c();
        ci.Name = 'Checklist Item Test';
        ci.Done__c = true;
        ci.Project_Phase__c = phase.Id;
        insert ci;

        fferpcore__Company__c c = new fferpcore__Company__c();
        c.Name = 'Test Company';
        insert c;

        Material_Group__c mg = new Material_Group__c();
        mg.Name = 'Test Group';
        insert mg;

        Material__c m = new Material__c();
        m.Name = 'Test Material';
        m.Project__c = p.Id;
        m.Quantity__c = 10;
        m.Unit_of_Measurement__c = 'ft';
        m.Company__c = c.Id;
        m.Material_Group__c = mg.Id;
        insert m;

        Date d = Date.Today();
        String s = 'Test Case Project';
        Test.StartTest();
        CPFTIncludingChecklistItemsAndMaterials.callCPFTIncludingChecklistItemsAndMaterials(p.Id, d, s);
        Test.StopTest();
		
        Checklist_Item__c checkItem = [SELECT Name, Done__c, Project_Phase__r.Name, 
        	Project_Phase__r.pse__Project__r.Name, Project_phase__r.pse__Project__r.pse__Start_Date__c 
              FROM Checklist_Item__c WHERE Project_Phase__r.pse__Project__r.Name ='Test Case Project' 
              LIMIT 1];

        
        //Check Checklist Item fields and successfully created Project and related fields
        System.assertEquals(false, checkItem.Done__c);
        System.assertEquals('Checklist Item Test', checkItem.Name);
        System.assertEquals('Phase Test',checkItem.Project_Phase__r.Name);
        System.assertEquals(s, checkItem.Project_Phase__r.pse__Project__r.Name);
        System.assertEquals(d, checkItem.Project_phase__r.pse__Project__r.pse__Start_Date__c);

        Material__c checkMaterial = [SELECT Name, Company__r.Name, Material_Group__r.Name, 
            Quantity__c, Unit_of_Measurement__c, Project__r.Name FROM Material__c WHERE 
            Project__r.Name = 'Test Case Project' LIMIT 1];

        //Check Material fields
        System.assertEquals('Test Company', checkMaterial.Company__r.Name);
        System.assertEquals('Test Group', checkMaterial.Material_Group__r.Name);
        System.assertEquals('Test Material', checkMaterial.Name);
        System.assertEquals(10, checkMaterial.Quantity__c);
        System.assertEquals('ft', checkMaterial.Unit_of_Measurement__c);
        System.assertEquals(s, checkMaterial.Project__r.Name);
    }
}
````