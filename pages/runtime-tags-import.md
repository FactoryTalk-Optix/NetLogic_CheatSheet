# Runtime import of PLC tags

Importing of PLC tags while the application is running is currently only supported for:

- TwinCAT
- RA Ethernet/IP
- Siemens S7 Profinet

### Warning

when importing tags at runtime, especially if filtering is used, a flat list is returned, this means that of you have a structure like:

- PLC
    - Motor1 (UDT)
        - Speed (Int32)
        - Torque (Int32)
    - Motor2 (UDT)
        - Speed (Int32)
        - Torque (Int32)

And you use the filtering mechanism to extract the `Speed` property of both structures, you will get:

- PLC
    - Speed (Int32)
    - Speed (Int32)

So, the whole structure should be imported or unique names shall be used

### Code example

The following code contains a method for each of the driver where runtime import is supported

```csharp
#region StandardUsing
using System;
using System.Collections.Generic;
using System.Linq;
using System.Reflection;
using FTOptix.CommunicationDriver;
using FTOptix.HMIProject;
using UAManagedCore;
#endregion

public class RuntimeTagsImport : FTOptix.NetLogic.BaseNetLogic
{
    /// <summary>
    /// Import tags from TwinCAT driver
    /// </summary>
    [FTOptix.NetLogic.ExportMethod]
    public static void ImportFromTwincat()
    {
        Log.Info(MethodBase.GetCurrentMethod().Name, "Starting runtime tags import");

        var station = Project.Current.Get<FTOptix.TwinCAT.Station>("CommDrivers/TwinCATDriver1/TwinCATStation1");

        // Call Browse method to retrieve data from PLC
        // prototypes parameter for TwinCat is always empty
        station.Browse(out var plcItems, out var prototypes);

        Log.Info("RuntimeTagsImport.ImportFromTwincat", "Fetched " + plcItems.Length + " tags");

        // Filter tag to import by tag name
        var tagInstances = FilterTagsToImport(new List<string>() { "SampleTag.A", "SomeOtherTag.B" }, plcItems); //TODO

        // Delete existing tags
        Log.Info("RuntimeTagsImport.ImportFromTwincat", "Clearing existing tags");
        station.Tags.Clear();

        Log.Info("RuntimeTagsImport.ImportFromTwincat", "Importing " + tagInstances.Length + " tags");

        // Filter tag to import by prototypeId
        // var tagInstances = GetTagsOfType("MachineParameter", plcItems);

        station.Import(tagInstances, prototypes);

        // Remote read on imported tag
        ReadTag("SampleTag.A", station);

        // Remote write on imported structure
        // WriteStructure("SomeOtherTag.B", station);

        // Remote read on imported structure
        // ReadStructure("SomeOtherTag.B", station);
    }

    /// <summary>
    /// Import tags from Profinet driver
    /// </summary>
    [FTOptix.NetLogic.ExportMethod]
    public static void ImportFromProfinet()
    {
        Log.Info(MethodBase.GetCurrentMethod().Name, "Starting runtime tags import");

        var station = Project.Current.Get<FTOptix.S7TiaProfinet.Station>("CommDrivers/S7TIAPROFINETDriver1/S7TIAPROFINETStation1");

        // Call Browse method to retrieve data from PLC
        // prototypes parameter for S7TiaProfinet is always empty
        station.Browse(out var plcItems, out var prototypes);

        Log.Info("RuntimeTagsImport.ImportFromProfinet", "Fetched " + plcItems.Length + " PLC structures");

        // Profinet tag are grouped by project, so we need to retrieve tags from root
        var root = plcItems.OfType<BasePlcItem>().ToList();
        // Root node is always the first element in the list
        // This is called "PLC" when browsing from the TagImporter
        var tags = root[0].Items;
        Log.Info("RuntimeTagsImport.ImportFromProfinet", "Fetched " + tags.Length + " tags");
        // Filter tag to import by tag name
        var tagInstances = FilterTagsToImport(new List<string>() { "PLC/TestFix", "PLC/TestDT" }, tags);

        // Delete existing tags
        Log.Info("RuntimeTagsImport.ImportFromProfinet", "Clearing existing tags");
        station.Tags.Clear();

        Log.Info("RuntimeTagsImport.ImportFromProfinet", "Importing " + tagInstances.Length + " tags");

        //Filter tag to import by prototypeId
        //var tagInstances = GetTagsOfType("MachineParameter", tags);

        station.Import(tagInstances, prototypes);
        Log.Info("Imported " + tagInstances.Length + " tags");

        // Remote read on imported tag
        ReadTag("SampleTag.A", station);

        // Remote write on imported structure
        // WriteStructure("SomeOtherTag.B", station);

        // Remote read on imported structure
        // ReadStructure("SomeOtherTag.B", station);
    }

    /// <summary>
    /// Import tags from RA Ethernet/IP driver
    /// </summary>
    [FTOptix.NetLogic.ExportMethod]
    public static void ImportFromRAEthernetIP()
    {
        Log.Info("RuntimeTagsImport.ImportFromRAEthernetIP", "Starting runtime tags import");

        var station = Project.Current.Get<FTOptix.RAEtherNetIP.Station>("CommDrivers/RAEtherNet_IPDriver1/RAEtherNet_IPStation1");

        // Call Browse method to retrieve data from PLC
        station.Browse(out var plcItems, out var prototypes);

        Log.Info("RuntimeTagsImport.ImportFromRAEthernetIP", "Fetched " + plcItems.Length + " tags structures from the PLC");

        // Filter tag to import by tag name
        var tagInstances = FilterTagsToImport(new List<string>() { "Controller Tags/Tank1/Level", "Controller Tags/Tank2/Level" }, plcItems);

        // Delete existing tags
        Log.Info("RuntimeTagsImport.ImportFromRAEthernetIP", "Clearing existing tags");
        station.Tags.Clear();

        Log.Info("RuntimeTagsImport.ImportFromRAEthernetIP", "Importing " + tagInstances.Length + " tags");

        if (tagInstances.Length == 0)
        {
            Log.Warning("RuntimeTagsImport.ImportFromRAEthernetIP", "No tags to import. Returning.");
            return;
        }

        //Filter tag to import by prototypeId
        // var tagInstances = GetTagsOfType("MachineParameter", plcItems);

        station.Import(tagInstances, prototypes);

        // Remote read on imported tag
        ReadTag("Controller Tags/Tank1/Level", station);

        // Remote write on imported structure
        // WriteStructure("Controller Tags/Tank1/Level", station);

        // Remote read on imported structure
        // ReadStructure("Controller Tags/Tank1", station);
    }

    /// <summary>
    /// Filter tags to import by tag name.
    /// Expects a list of tag paths to import (like "Controller Tags/Motor1/Speed" and
    /// the list of all tags that were fetch from the PLC.
    /// If the list of tags to import is empty, all fetched tags will be returned.
    /// Warning: a flat list is returned, so the tag paths must be unique.
    /// </summary>
    /// <param name="tagPathsToImport">List of tag paths to import (path with forward slashes "/")</param>
    /// <param name="plcItems">List of all tags fetched from the PLC</param>
    /// <returns>List of tags (Struct[]) that matches the input list</returns>
    private static Struct[] FilterTagsToImport(List<string> tagPathsToImport, Struct[] plcItems)
    {
        var listOfFoundTags = new List<Struct>();

        // If no list of tags to import was provided, return all tags
        if (tagPathsToImport.Count == 0)
        {
            Log.Warning("RuntimeTagsImport.FilterTagsToImport", "No list of tags to import was provided. Returning all tags.");
            return plcItems;
        }

        if (plcItems.Length == 0)
        {
            Log.Warning("RuntimeTagsImport.FilterTagsToImport", "No tags were fetched from the PLC. Returning empty list.");
            return plcItems;
        }

        // Filter tags to import by tag name
        foreach (string tagPath in tagPathsToImport)
        {
            string[] splitPath = tagPath.Split('/');
            var node = plcItems;
            for (int i = 0; i < splitPath.Length; i++)
            {
                for (int k = 0; k < node.Length; k++)
                {
                    if (((BasePlcItem)node[k]).Name == splitPath[i])
                    {
                        if (i == splitPath.Length - 1)
                        {
                            listOfFoundTags.Add(node[k]);
                            break;
                        }
                        else
                        {
                            node = ((BasePlcItem)node[k]).Items;
                            break;
                        }
                    }
                }
            }
        }

        return listOfFoundTags.ToArray();
    }

    /// <summary>
    /// Performs a remote read on a tag.
    /// </summary>
    /// <param name="tagName"></param>
    /// <param name="station"></param>
    private static void ReadTag(string tagName, FTOptix.CommunicationDriver.CommunicationStation station)
    {
        try
        {
            var tag = station.Get<FTOptix.CommunicationDriver.Tag>("Tags/" + tagName);
            var result = tag.RemoteRead();
            Log.Info($"RemoteRead succeed -> {tagName}: {result.Value}");
        }
        catch (Exception e)
        {
            Log.Error("RemoteRead failed: " + e.Message);
        }
    }
}


```