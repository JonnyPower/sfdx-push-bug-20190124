# SFDX Push Issue with multiple package project

In this project we have a package (package-01) that deploys the custom object `Example_Object__c` with a field `First_Field__c` .

Our second package (package-02) depends on the first, and deploys a second field: `Second_Field__c`. 

SFDX does not correctly merge the complimentary metadata used for a full push of the umbrella project.

- Convert the source to mdapi metadata

```
sfdx force:source:convert
```

- Open the package.xml and see that it has only generated the package.xml and `src` folder based on package-01 - maybe because it is the default project.

package.xml:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Package xmlns="http://soap.sforce.com/2006/04/metadata">
  <types>
    <name>CustomObject</name>
    <members>Example_Object__c</members>
  </types>
  <types>
    <name>CustomField</name>
    <members>Example_Object__c.First_Field__c</members>
  </types>
  <version>44.0</version>
</Package>
```

object.xml:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<CustomObject xmlns="http://soap.sforce.com/2006/04/metadata">
    <actionOverrides>
        <actionName>Accept</actionName>
        <type>Default</type>
    </actionOverrides>
    <actionOverrides>
        <actionName>CancelEdit</actionName>
        <type>Default</type>
    </actionOverrides>
    <actionOverrides>
        <actionName>Clone</actionName>
        <type>Default</type>
    </actionOverrides>
    <actionOverrides>
        <actionName>Delete</actionName>
        <type>Default</type>
    </actionOverrides>
    <actionOverrides>
        <actionName>Edit</actionName>
        <type>Default</type>
    </actionOverrides>
    <actionOverrides>
        <actionName>List</actionName>
        <type>Default</type>
    </actionOverrides>
    <actionOverrides>
        <actionName>New</actionName>
        <type>Default</type>
    </actionOverrides>
    <actionOverrides>
        <actionName>SaveEdit</actionName>
        <type>Default</type>
    </actionOverrides>
    <actionOverrides>
        <actionName>Tab</actionName>
        <type>Default</type>
    </actionOverrides>
    <actionOverrides>
        <actionName>View</actionName>
        <type>Default</type>
    </actionOverrides>
    <allowInChatterGroups>false</allowInChatterGroups>
    <compactLayoutAssignment>SYSTEM</compactLayoutAssignment>
    <deploymentStatus>Deployed</deploymentStatus>
    <enableActivities>false</enableActivities>
    <enableBulkApi>true</enableBulkApi>
    <enableFeeds>false</enableFeeds>
    <enableHistory>false</enableHistory>
    <enableReports>false</enableReports>
    <enableSearch>false</enableSearch>
    <enableStreamingApi>true</enableStreamingApi>
    <label>Example Object</label>
    <nameField>
        <displayFormat>EO-{00000000}</displayFormat>
        <label>Name</label>
        <type>AutoNumber</type>
    </nameField>
    <pluralLabel>Example Objects</pluralLabel>
    <searchLayouts/>
    <sharingModel>ReadWrite</sharingModel>
    <visibility>Public</visibility>
    <fields>
        <fullName>First_Field__c</fullName>
        <externalId>false</externalId>
        <label>Alias</label>
        <length>255</length>
        <required>false</required>
        <trackTrending>false</trackTrending>
        <type>Text</type>
        <unique>false</unique>
    </fields>
</CustomObject>
```

- Convert the source for only package-02

```
sfdx force:source:convert -r package-02
```

- Open the package.xml and see that it has correctly generated the metadata deployment for just the custom field.

package.xml:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Package xmlns="http://soap.sforce.com/2006/04/metadata">
  <types>
    <name>CustomField</name>
    <members>Example_Object__c.Second_Field__c</members>
  </types>
  <version>44.0</version>
</Package>
```

object.xml:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<CustomObject xmlns="http://soap.sforce.com/2006/04/metadata">
    <fields>
        <fullName>Second_Field__c</fullName>
        <externalId>false</externalId>
        <label>Alias</label>
        <length>255</length>
        <required>false</required>
        <trackTrending>false</trackTrending>
        <type>Text</type>
        <unique>false</unique>
    </fields>
</CustomObject>
```

- Do a push and observe that it errors out;

```
sfdx force:source:push -u <yourusername>
```

```
PROJECT PATH  ERROR
────────────  ─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
N/A           An object 'Example_Object__c.Second_Field__c' of type CustomField was named in package.xml, but was not found in zipped directory
```

- **Based on the above it seems that the package.xml is correctly generated for the push, but the object.xml is not merged with all the packages.**

- If you deploy one subfolder at a time then you can deploy to your org, except this is problematic for common development usecases such as continuous integration or use in IDEs (e.g. Illuminated Cloud uses the underlying push) 
	
	- On CI: CI necessitates being able to do a check-only deployment to keep the org clean for future CI jobs, having to fully deploy dependencies first breaks this and makes CI much harder. Must resort to building logic to generate merged metadata ourselves.
	
```
sfdx force:source:deploy -p package-01 -u <username>
```

```
=== Deployed Source
FULL NAME                         TYPE          PROJECT PATH
────────────────────────────────  ────────────  ────────────────────────────────────────────────────────────────────────────────────────────────
Example_Object__c                 CustomObject  package-01\force-app\main\default\objects\Example_Object__c\Example_Object__c.object-meta.xml
Example_Object__c.First_Field__c  CustomField   package-01\force-app\main\default\objects\Example_Object__c\fields\First_Field__c.field-meta.xml
```

```
sfdx force:source:deploy -p package-02 -u <username>
```

```
=== Deployed Source
FULL NAME                          TYPE         PROJECT PATH
─────────────────────────────────  ───────────  ─────────────────────────────────────────────────────────────────────────────────────────────────
Example_Object__c.Second_Field__c  CustomField  package-02\force-app\main\default\objects\Example_Object__c\fields\Second_Field__c.field-meta.xml
```