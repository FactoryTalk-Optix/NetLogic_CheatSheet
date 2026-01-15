# Design time import of PLC tags

>[!NOTE]
> This feature is available starting from FactoryTalk Optix 1.7.x

## TwinCAT

To import tags from a TwinCAT PLC, you can use the `TwinCAT` driver. The import process allows you to filter tags based on specific criteria, such as tag name or data type.

>[!NOTE]
> As per FactoryTalk Optix 1.7.x, only the TwinCAT driver supports online and offline import of tags via NetLogic.

```csharp
#region Using directives
using System;
using UAManagedCore;
using OpcUa = UAManagedCore.OpcUa;
using FTOptix.UI;
using FTOptix.HMIProject;
using FTOptix.NativeUI;
using FTOptix.Retentivity;
using FTOptix.CoreBase;
using FTOptix.Core;
using FTOptix.NetLogic;
using FTOptix.TwinCAT;
using FTOptix.CommunicationDriver;
using static UAManagedCore.TagImporterAPI;
using System.Threading.Tasks;
#endregion

public class DesignTimeNetLogic1 : BaseNetLogic
{
    [ExportMethod]
    public void OfflineImport()
    {
        // First get the existing TwinCAT station from the project
        var station = Project.Current.Get<FTOptix.TwinCAT.Station>("CommDrivers/TwinCATDriver1/TwinCATStation1");

        // Then get the tag importer node from the station
        var tagImporter = station.TagImporter;

        // Set the file path to the TwinCAT TPY file you want to import
        tagImporter.FilePath = ResourceUri.FromAbsoluteFilePath("D:\\test\\Optix\\Tools\\TwinCAT\\tpy\\AllSimpleType.tpy");
        tagImporter.Mode = FetchMode.Offline;

        // Fire and forget, import is run in a background thread
        _ = tagImporter.Import();
    }

    [ExportMethod]
    public void OfflineImportWithContinuation()
    {
        var station = Project.Current.Get<FTOptix.TwinCAT.Station>("CommDrivers/TwinCATDriver1/TwinCATStation1");
        var tagImporter = station.TagImporter;

        tagImporter.FilePath = ResourceUri.FromAbsoluteFilePath("D:\\test\\Optix\\Tools\\TwinCAT\\tpy\\AllSimpleType.tpy");
        tagImporter.Mode = FetchMode.Offline;

        // Import is started in a background thread,
        // continuation is run in the current thread after the import is completed
        var task = tagImporter.Import(Filter);
        task.ContinueWith((t) =>
        {
            if (t.IsFaulted)
            {
                Log.Error("Import task failed: " + t.Exception?.GetBaseException().Message);
            }
            else if (t.IsCanceled)
            {
                Log.Info("Import task was canceled.");
            }
            else
            {
                Log.Info("Import task completed successfully.");
            }
        });
    }

    [ExportMethod]
    public void OnlineImport()
    {
        var station = Project.Current.Get<FTOptix.TwinCAT.Station>("CommDrivers/TwinCATDriver1/TwinCATStation1");
        var tagImporter = station.TagImporter;

        tagImporter.Mode = FetchMode.Online;
        _ = tagImporter.Import(Filter);
    }


    private bool Filter(string tag, string type)
    {
        // Filter by tag name
        return tag.Contains("int");

        // or

        // Filter by type
        // return type == "INT";
    }
}
```

An example of TPY file that can be used for testing:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<PlcProjectInfo>
   <ProjectInfo>
      <Path>C:\Users\asem\Desktop\TestTwinCAT\TwinCAT Project1\TwinCAT Project1\Untitled1\Untitled1.plcproj</Path>
      <ChangeDate>2018-02-11T04:39:01</ChangeDate>
      <Library>
         <Name>tc2_standard, 3.3.2.0 (beckhoff automation gmbh)</Name>
      </Library>
      <Library>
         <Name>tc2_system, 3.4.17.0 (beckhoff automation gmbh)</Name>
      </Library>
      <Library>
         <Name>tc3_module, 3.3.17.0 (beckhoff automation gmbh)</Name>
      </Library>
   </ProjectInfo>
   <RoutingInfo>
      <AdsInfo>
         <NetId>172.17.17.35.1.1</NetId>
         <Port>851</Port>
         <TargetName>Untitled1</TargetName>
      </AdsInfo>
   </RoutingInfo>
   <CompilerInfo>
      <CpuFamily>Intel X86</CpuFamily>
      <CompilerVersion>3.5.10.20</CompilerVersion>
      <TwinCATVersion>3.4.0.0</TwinCATVersion>
      <TCatPlcCtrlVersion>3.4.0.0</TCatPlcCtrlVersion>
   </CompilerInfo>
   <TargetInfo>
      <CpuFamily>Intel X86</CpuFamily>
      <DataAreaInfo>
         <DataSize>10485760</DataSize>
         <RetainSize>128000</RetainSize>
         <MAreaSize>128000</MAreaSize>
         <InputSize>128000</InputSize>
         <OutputSize>128000</OutputSize>
      </DataAreaInfo>
   </TargetInfo>
   <DataTypes>
   </DataTypes>
   <Symbols>
      <Symbol>
         <Name>MAIN.VarBit</Name>
         <Type>BIT</Type>
         <IGroup>16448</IGroup>
         <IOffset>512508</IOffset>
         <BitSize>8</BitSize>
      </Symbol>
      <Symbol>
         <Name>MAIN.VarDateAndTime</Name>
         <Type>DATE_AND_TIME</Type>
         <IGroup>16448</IGroup>
         <IOffset>512512</IOffset>
         <BitSize>32</BitSize>
      </Symbol>
      <Symbol>
         <Name>MAIN.VarLTime</Name>
         <Type>LTIME</Type>
         <IGroup>16448</IGroup>
         <IOffset>512520</IOffset>
         <BitSize>64</BitSize>
      </Symbol>
      <Symbol>
         <Name>MAIN.VarTime</Name>
         <Type>TIME</Type>
         <IGroup>16448</IGroup>
         <IOffset>512528</IOffset>
         <BitSize>32</BitSize>
      </Symbol>
      <Symbol>
         <Name>MAIN.VarTod</Name>
         <Type>TOD</Type>
         <IGroup>16448</IGroup>
         <IOffset>512532</IOffset>
         <BitSize>32</BitSize>
      </Symbol>
      <Symbol>
         <Name>MAIN.VarDate</Name>
         <Type>DATE</Type>
         <IGroup>16448</IGroup>
         <IOffset>512536</IOffset>
         <BitSize>32</BitSize>
      </Symbol>
      <Symbol>
         <Name>MAIN.VarDT</Name>
         <Type>DT</Type>
         <IGroup>16448</IGroup>
         <IOffset>512540</IOffset>
         <BitSize>32</BitSize>
      </Symbol>
      <Symbol>
         <Name>MAIN.VarBool</Name>
         <Type>BOOL</Type>
         <IGroup>16448</IGroup>
         <IOffset>512594</IOffset>
         <BitSize>8</BitSize>
      </Symbol>
      <Symbol>
         <Name>MAIN.VarByte</Name>
         <Type>BYTE</Type>
         <IGroup>16448</IGroup>
         <IOffset>512595</IOffset>
         <BitSize>8</BitSize>
      </Symbol>
      <Symbol>
         <Name>MAIN.VarDint</Name>
         <Type>DINT</Type>
         <IGroup>16448</IGroup>
         <IOffset>536300</IOffset>
         <BitSize>32</BitSize>
      </Symbol>
      <Symbol>
         <Name>MAIN.VarDword</Name>
         <Type>DWORD</Type>
         <IGroup>16448</IGroup>
         <IOffset>536304</IOffset>
         <BitSize>32</BitSize>
      </Symbol>
      <Symbol>
         <Name>MAIN.VarInt</Name>
         <Type>INT</Type>
         <IGroup>16448</IGroup>
         <IOffset>536308</IOffset>
         <BitSize>16</BitSize>
      </Symbol>
      <Symbol>
         <Name>MAIN.VarSint</Name>
         <Type>SINT</Type>
         <IGroup>16448</IGroup>
         <IOffset>536310</IOffset>
         <BitSize>8</BitSize>
      </Symbol>
      <Symbol>
         <Name>MAIN.VarUsint</Name>
         <Type>USINT</Type>
         <IGroup>16448</IGroup>
         <IOffset>536311</IOffset>
         <BitSize>8</BitSize>
      </Symbol>
      <Symbol>
         <Name>MAIN.VarLint</Name>
         <Type>LINT</Type>
         <IGroup>16448</IGroup>
         <IOffset>536312</IOffset>
         <BitSize>64</BitSize>
      </Symbol>
      <Symbol>
         <Name>MAIN.VarLreal</Name>
         <Type>LREAL</Type>
         <IGroup>16448</IGroup>
         <IOffset>536320</IOffset>
         <BitSize>64</BitSize>
      </Symbol>
      <Symbol>
         <Name>MAIN.VarLWord</Name>
         <Type>LWORD</Type>
         <IGroup>16448</IGroup>
         <IOffset>536328</IOffset>
         <BitSize>64</BitSize>
      </Symbol>
      <Symbol>
         <Name>MAIN.VarReal</Name>
         <Type>REAL</Type>
         <IGroup>16448</IGroup>
         <IOffset>536336</IOffset>
         <BitSize>32</BitSize>
      </Symbol>
      <Symbol>
         <Name>MAIN.VarUint</Name>
         <Type>UINT</Type>
         <IGroup>16448</IGroup>
         <IOffset>536422</IOffset>
         <BitSize>16</BitSize>
      </Symbol>
      <Symbol>
         <Name>MAIN.VarTimOfDay</Name>
         <Type>TIME_OF_DAY</Type>
         <IGroup>16448</IGroup>
         <IOffset>536424</IOffset>
         <BitSize>32</BitSize>
      </Symbol>
      <Symbol>
         <Name>MAIN.VarUdint</Name>
         <Type>UDINT</Type>
         <IGroup>16448</IGroup>
         <IOffset>536428</IOffset>
         <BitSize>32</BitSize>
      </Symbol>
      <Symbol>
         <Name>MAIN.VarUlint</Name>
         <Type>ULINT</Type>
         <IGroup>16448</IGroup>
         <IOffset>536432</IOffset>
         <BitSize>64</BitSize>
      </Symbol>
      <Symbol>
         <Name>MAIN.VarWord</Name>
         <Type>WORD</Type>
         <IGroup>16448</IGroup>
         <IOffset>536440</IOffset>
         <BitSize>16</BitSize>
      </Symbol>
      <Symbol>
         <Name>MAIN.VarWstring</Name>
         <Type>WSTRING(80)</Type>
         <IGroup>16448</IGroup>
         <IOffset>536442</IOffset>
         <BitSize>1296</BitSize>
      </Symbol>
      <Symbol>
         <Name>MAIN.VarString</Name>
         <Type Decoration="40000050">STRING(80)</Type>
         <IGroup>16448</IGroup>
         <IOffset>537880</IOffset>
         <BitSize>648</BitSize>
      </Symbol>
	</Symbols>
</PlcProjectInfo>
```
