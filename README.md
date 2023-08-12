# APICv10-Persisting-Lookup-Property-Config-Files-on-the-Gateway
This document is to go over how to add and persist lookup (js require / xsl:include) files into APIC Gateways, which code/scripts would use to reference variables for or call the file in general.

Uploading files directly to the APIC domain of the APIC gateway that your gatewayscript or xslt policies reference to pull configuration details into the code during runtime will eventually disappear on the next reboot of the gateway. To persist these files, you must either create configmaps (if on k8s or ocp) or gateway-extensions.
Configmaps will require reboots to the gateway every time there is a change, but gateway-extensions do not require reboot of the gateways, therefore this documentation will focus on that.

This document assumes that the implimentor:
- has privileges to upload gateway-extensions to the CLoud Manager,
- knows how to develop apis on APIC,
- and knows what DataPower Gateway is and has a gateway environment to execute the tasks detailed in this document.

For this example, we have an API, which contains a GatewayScript that reads a file on the gateway to pull port information from a lookup properties config file.
Use cases like this exist to ensure there is one externalized source for multiple code/scripts to reference on different environments to achieve portability of code and lessen having to change certain values in multiple apis many times when a change occurs.

The following gatewayscript uses the fs module to read the file local:///lookup_files/lookup_file_config.json  
The api the contains the gatewayscript may be downloaded from [lookup-file.yaml](https://github.com/ibmArtifacts/APICv10-Persisting-Lookup-Property-Config-Files-on-the-Gateway/blob/main/lookup-file.yaml).
```  
var fs = require('fs');

fs.readFile("local:///lookup_files/lookup_file_config.json", function(error,data) {
    if(error) {
        // Handle error.
    } else {
        
        //data is the content in local:///lookup_files/lookup_file_config.json
        var vLookup_file = JSON.parse(data);
        
        //Get the current catalog during runtime.
        var vCurrentCatalog = context.get('api.catalog.name');
        
        //Lookup the port configuration from the config file.
        var vPort = vLookup_file[context.get('api.catalog.name')].port
        
        context.message.body.write(vPort);
    }
}
);
```
The api only will work if the lookup file lookup_file_config.json is persisted on the gateway file system in the local:///lookup_files directory.
Here are the instructions to persist the file with a gateway-extension policy.
## Getting the DataPower export with the lookup file  

1. Navigate to and log into a gateway to create the directory and upload the file to the directory. In our example, it will be local:///lookup_files
   
![image](https://github.com/ibmArtifacts/APICv10-Persisting-Lookup-Property-Config-Files-on-the-Gateway/assets/66093865/8bc19352-d634-40c7-af99-94f0f7677a29)

2. Go to the export section and use "Export configuration and files from the current domain", select next, update the Export file name and choose "Export all local files", select next again, and Download the export.

![image](https://github.com/ibmArtifacts/APICv10-Persisting-Lookup-Property-Config-Files-on-the-Gateway/assets/66093865/727a60ed-d180-43e6-b395-9fb757a30b6d)  

3. Once exported, unzip the file to clean up any directories or files not required (e.g. config, dp-aux, local/extension, local/gwapi, local/isp, local/config-sequence.cfg, etc).  
![image](https://github.com/ibmArtifacts/APICv10-Persisting-Lookup-Property-Config-Files-on-the-Gateway/assets/66093865/10440aee-676c-4cd5-ba48-5c6aca237d48)

4. Once removed, navigate back to the level where the export.xml file is located, and rezip the export.  
![image](https://github.com/ibmArtifacts/APICv10-Persisting-Lookup-Property-Config-Files-on-the-Gateway/assets/66093865/95b8b44f-5ea2-4992-8266-a74df00c7825)

[Here is a sample export where the lookup file is included](https://github.com/ibmArtifacts/APICv10-Persisting-Lookup-Property-Config-Files-on-the-Gateway/blob/main/export-files.zip).  
NOTE: If you are to use the export-files.zip, ensure you update the export.xml to include the file name and directory you're trying to upload to the gateway. In addition, ensure you have the file and directory present in the zip file.  

## Creating the Gateway-Extension to push the export to all the gateways  

5. In the same place you have the gateway export (e.g. export-files.zip), create a file titled manifest.json, and include the following sample details in the file. Then zip up the gateway export and manifest.json file together naming the zip gateway-extension.zip.
This tells APIC to import the export-files.zip to the gateways associated to the topology you upload the Gateway-Extension to.  
```
{
    "extension": {
      "properties": {
        "deploy-policy-emulator": false
      },
      "files": [
        {
          "filename":"export-files.zip",
          "deploy": "immediate",
          "type": "dp-import"
        }
    ]
  }   
}
```  
More details on the manifest properties may be found in the [IBM documentation for Gateway extension manifest](https://www.ibm.com/docs/en/api-connect/10.0.5.x_lts?topic=gateway-extensions-manifest).  

![image](https://github.com/ibmArtifacts/APICv10-Persisting-Lookup-Property-Config-Files-on-the-Gateway/assets/66093865/c63f5c88-84a2-41ad-80b3-f657763ec5e6)  

At this point, you may remove the file and directory originally put in the apic domain to take the export, because you will want to see if the next steps that imports the file and directory works.

## Upload the Gateway-Extension
6. Log into the Cloud Manager, and navigate to the Topology section. Then located the gateway service and click on the elipsis at the end of the gateway to select Configure gateway.  
![image](https://github.com/ibmArtifacts/APICv10-Persisting-Lookup-Property-Config-Files-on-the-Gateway/assets/66093865/62e724cf-f1d9-49aa-aebc-9f9a39e6eba2)

7. Click Add and upload the gateway-extension.zip.  
8. Afterwards, you should see the directory and file on the gateway.  
![image](https://github.com/ibmArtifacts/APICv10-Persisting-Lookup-Property-Config-Files-on-the-Gateway/assets/66093865/81a8cc67-66fe-4e5f-9b53-bb8cf17bc340)








