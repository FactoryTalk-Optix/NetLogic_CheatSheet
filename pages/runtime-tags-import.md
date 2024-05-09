# Runtime import of PLC tags

Importing of PLC tags while the application is running is currently only supported for:

- TwinCAT
- RA Ethernet/IP
- Siemens S7 Profinet

The following code contains a method for each of these drivers

```csharp
#region StandardUsing
using System;
using FTOptix.CoreBase;
using FTOptix.HMIProject;
using UAManagedCore;
using FTOptix.CommunicationDriver;
using OpcUa = UAManagedCore.OpcUa;
using FTOptix.NetLogic;
using System.Collections.Generic;
using System.Linq;
using System.Reflection;
#endregion

public class RuntimeTagsImport : FTOptix.NetLogic.BaseNetLogic
{
    [FTOptix.NetLogic.ExportMethod]
    public static void ImportFromTwincat()
    {
        Log.Info(MethodBase.GetCurrentMethod().Name, "Starting runtime tags import");

        var station = Project.Current.Get<FTOptix.TwinCAT.Station>("CommDrivers/TwinCATDriver1/TwinCATStation1");

        // Call Browse method to retrieve data from PLC
        // prototypes parameter for TwinCat is always empty
        station.Browse(out Struct[] plcItems, out Struct[] prototypes);

        Log.Info(MethodBase.GetCurrentMethod().Name, "Fetched " + plcItems.Length + " tags");

        // Filter tag to import by tag name
        var tagInstances = GetItemsToImport(new List<string>() { "SampleTag.A", "SomeOtherTag.B" }, plcItems);

        Log.Info(MethodBase.GetCurrentMethod().Name, "Importing " + tagInstances.Length + " tags");

        //Filter tag to import by prototypeId
        //var tagInstances = GetTagsOfType("MachineParameter", plcItems);

        station.Import(tagInstances, prototypes);

        // Remote read on imported tag
        ReadTag("SampleTag.A", station);

        //// Remote write on imported structure
        WriteStructure("SomeOtherTag.B", station);

        // Remote read on imported structure
        ReadStructure("SomeOtherTag.B", station);
    }

    [FTOptix.NetLogic.ExportMethod]
    public static void ImportFromProfinet()
    {
        Log.Info(MethodBase.GetCurrentMethod().Name, "Starting runtime tags import");

        var station = Project.Current.Get<FTOptix.S7TiaProfinet.Station>("CommDrivers/S7TIAPROFINETDriver1/S7TIAPROFINETStation1");

        // Call Browse method to retrieve data from PLC
        // prototypes parameter for S7TiaProfinet is always empty
        station.Browse(out Struct[] plcItems, out Struct[] prototypes);

        Log.Info(MethodBase.GetCurrentMethod().Name, "Fetched " + plcItems.Length + " tags");

        // Profinet tag are grouped by project, so we need to retrieve tags from root
        var root = plcItems.OfType<BasePlcItem>().ToList();
        var tags = root[0].Items;
        Log.Info("Fetched " + tags.Length + " tags");
        // Filter tag to import by tag name
        var tagInstances = GetItemsToImport(new List<string>() { "SampleTag.A", "SomeOtherTag.B" }, tags);

        Log.Info(MethodBase.GetCurrentMethod().Name, "Importing " + tagInstances.Length + " tags");

        //Filter tag to import by prototypeId
        //var tagInstances = GetTagsOfType("MachineParameter", tags);

        station.Import(tagInstances, prototypes);
        Log.Info("Imported " + tagInstances.Length + " tags");

        // Remote read on imported tag
        ReadTag("SampleTag.A", station);

        // Remote write on imported structure
        WriteStructure("SomeOtherTag.B", station);

        // Remote read on imported structure
        ReadStructure("SomeOtherTag.B", station);
    }

    /*

    UNTESTED CODE

    [FTOptix.NetLogic.ExportMethod]
    public static void ImportFromRAEthernetIP()
    {
        Log.Info(MethodBase.GetCurrentMethod().Name, "Starting runtime tags import");

        var station = Project.Current.Get<FTOptix.RAEtherNetIP.Station>("CommDrivers/RAEtherNet_IPDriver1/RAEtherNet_IPStation1");

        // Call Browse method to retrieve data from PLC
        station.Browse(out Struct[] plcItems, out Struct[] prototypes);

        Log.Info(MethodBase.GetCurrentMethod().Name, "Fetched " + plcItems.Length + " tags");

        // Filter tag to import by tag name
        var tagInstances = GetItemsToImport(new List<string>() { "SampleTag.A", "SomeOtherTag.B" }, plcItems);

        Log.Info(MethodBase.GetCurrentMethod().Name, "Importing " + tagInstances.Length + " tags");

        //Filter tag to import by prototypeId
        //var tagInstances = GetTagsOfType("MachineParameter", plcItems);

        station.Import(tagInstances, prototypes);

        // Remote read on imported tag
        ReadTag("SampleTag.A", station);

        //// Remote write on imported structure
        WriteStructure("SomeOtherTag.B", station);

        // Remote read on imported structure
        ReadStructure("SomeOtherTag.B", station);
    }

    */

    private static Struct[] GetItemsToImport(List<string> tagToImport, Struct[] plcItems)
    {
        var result = new List<Struct>();
        var basePlcItems = plcItems.OfType<BasePlcItem>().ToList();

        foreach (var tagName in tagToImport)
        {
            var plcItemTag = basePlcItems.First(basePlcItem => basePlcItem.Name == tagName);
            if (plcItemTag != null)
                result.Add(plcItemTag);
        }

        return result.ToArray();
    }

    private static Struct[] GetTagsOfType(string prototypeName, Struct[] plcItems)
    {
        var tagStructures = plcItems.OfType<StructureInfo>().ToList();
        var result = tagStructures.Where(item => item.PrototypeId == prototypeName);
        return result.ToArray();
    }

    private static void ReadTag(string tagName, FTOptix.CommunicationDriver.CommunicationStation station)
    {
        try
        {
            var tag = station.Get<FTOptix.CommunicationDriver.Tag>("Tags/" + tagName);
            var result = tag.RemoteRead();
            Log.Info($"RemoteRead succeded -> {tagName}: {result.Value}");
        }
        catch (Exception e)
        {
            Log.Error("RemoteRead failed: " + e.Message);
        }
    }

    private static void ReadStructure(string structName, FTOptix.CommunicationDriver.CommunicationStation station)
    {
        var variables = new List<RemoteChildVariable>()
        {
            new RemoteChildVariable("ChildrenVarA"),
            new RemoteChildVariable("ChildrenVarB"),
            new RemoteChildVariable("ChildrenVarC")
        };

        var tagStructure = station.Get<FTOptix.CommunicationDriver.TagStructure>("Tags/" + structName);

        try
        {

            var values = tagStructure.ChildrenRemoteRead(variables).ToArray();

            Log.Info($"TagStructure ChildrenRemoteRead succeded -> Vent: {values[0].Value}");
            Log.Info($"TagStructure ChildrenRemoteRead succeded -> Duration: {values[1].Value}");
            Log.Info($"TagStructure ChildrenRemoteRead succeded -> SetPoint_Temp: {values[2].Value}");
        }
        catch (Exception ex)
        {
            Log.Error("TagStructure ChildrenRemoteRead failed: " + ex.ToString());
        }
    }

    private static void WriteStructure(string structName, FTOptix.CommunicationDriver.CommunicationStation station)
    {
        Random rnd = new Random();

        var values = new List<RemoteChildVariableValue>()
        {
            new RemoteChildVariableValue("ChildrenVarA", true),
            new RemoteChildVariableValue("ChildrenVarB", 5000.0),
            new RemoteChildVariableValue("ChildrenVarC", (float)rnd.NextDouble()),
        };

        var tagStructure = station.Get<FTOptix.CommunicationDriver.TagStructure>("Tags/" + structName);

        try
        {
            tagStructure.ChildrenRemoteWrite(values);
        }
        catch (Exception ex)
        {
            Log.Error("TagStructure ChildrenRemoteWrite failed: " + ex.ToString());
        }
    }
}

```