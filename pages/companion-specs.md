# Companion Specifications

## Load Companion Specifications

- Companion Specifications files can be downloaded from [https://github.com/OPCFoundation/UA-Nodeset](https://github.com/OPCFoundation/UA-Nodeset)
- Add them to FT Optix in one of these paths (make sure to replace your username or the proper FT Optix setup folder):
    - `C:\Users\<username>\Documents\Rockwell Automation\FactoryTalk Optix\CompanionSpecifications`
    - `C:\Program Files\Rockwell Automation\FactoryTalk Optix\Studio <version number>\CompanionSpecifications`
- Restart the IDE after adding files to one of those paths in order to load new files
    - Check the FT Optix output if any error was reported after loading the files

## Create an instance of a Companion Specs type

Please make sure the proper XML file was added to FT Optix before executing the script

We will load an instance of the BlockType object from the DI specifications, the XML relevant part is:

```xml
<?xml version="1.0" encoding="utf-8"?>
    <UANodeSet LastModified="2023-04-13T14:49:52.318Z" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns="http://opcfoundation.org/UA/2011/03/UANodeSet.xsd" xmlns:uax="http://opcfoundation.org/UA/2008/02/Types.xsd" xmlns:si="http://www.siemens.com/OPCUA/2017/SimaticNodeSetExtensions" xmlns:xsd="http://www.w3.org/2001/XMLSchema" xmlns:s1="http://opcfoundation.org/UA/Machinery/ProcessValues/Types.xsd" xmlns:s2="http://opcfoundation.org/UA/PADIM/Types.xsd" xmlns:ua="http://unifiedautomation.com/Configuration/NodeSet.xsd">
    <NamespaceUris>
        <Uri>http://opcfoundation.org/UA/DI/</Uri>
    </NamespaceUris>

...

    <UAObjectType IsAbstract="true" NodeId="ns=1;i=1003" BrowseName="1:BlockType">
        <DisplayName>BlockType</DisplayName>
        <Description>Adds the concept of Blocks needed for block-oriented FieldDevices</Description>
        <Documentation>https://reference.opcfoundation.org/v105/DI/v103/docs/4.11</Documentation>
        <References>
            <Reference ReferenceType="HasSubtype" IsForward="false">ns=1;i=1001</Reference>
            <Reference ReferenceType="HasProperty">ns=1;i=6009</Reference>
            <Reference ReferenceType="HasProperty">ns=1;i=6010</Reference>
            <Reference ReferenceType="HasProperty">ns=1;i=6011</Reference>
            <Reference ReferenceType="HasProperty">ns=1;i=6012</Reference>
            <Reference ReferenceType="HasProperty">ns=1;i=6013</Reference>
        </References>
    </UAObjectType>
    
...
```

From the XML we can see that the numeric identifier for the BlockType is `1003`, now in FT Optix we need to read the NameSpaceIndex. We will also need the NamespaceUri for that CompanionSpecs, in this case is `http://opcfoundation.org/UA/DI/`

```csharp
[ExportMethod]
public void CreateDIBlockType()
{
    Log.Info("Create DI Block Type NetLogic", "Searching NameSpaceIndex for DI");
    var nsIndex = LogicObject.Context.GetNamespaceIndex("http://opcfoundation.org/UA/DI/");
    if (nsIndex == -1)
    {
        Log.Error("Create DI Block Type NetLogic", "NameSpaceIndex for DI not found, make sure the CompanionSpecification for DI was loaded");
        return;
    }
    else
    {
        Log.Info("Create DI Block Type NetLogic", "NameSpaceIndex for DI: " + nsIndex);
    }
    // The ID of the BlockType is 1003 - we can get this value from the DI XML file
    var blockTypeNodeId = new NodeId(nsIndex, 1003);
    // Create an instance of the element
    Log.Info("Create DI Block Type NetLogic", "BlockTypeNodeId: " + blockTypeNodeId);
    var blockTypeInstance = InformationModel.MakeObject("BlockTypeInstance", blockTypeNodeId);
    Project.Current.Get("Model/BlockTypeInstance")?.Delete();
    Project.Current.Get("Model").Add(blockTypeInstance);
    Log.Info("Create DI Block Type NetLogic", "BlockTypeInstance created and added to the model");
}
```