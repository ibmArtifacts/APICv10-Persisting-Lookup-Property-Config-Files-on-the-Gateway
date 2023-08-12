# APICv10-Persisting-Lookup-Property-Config-Files-on-the-Gateway
This document is to go over how to add and persist lookup (js require / xsl:include) files into APIC Gateways, which code/scripts would use to reference variables for or call the file in general.

Uploading files directly to the APIC domain of the APIC gateway that your gatewayscript or xslt policies reference to pull configuration details into the code during runtime will eventually disappear on the next reboot of the gateway. To persist these files, you must either create configmaps (if on k8s or ocp) or gateway-extensions.
Configmaps will require reboots to the gateway every time there is a change, but gateway-extensions do not require reboot of the gateways, therefore this documentation will focus on that.

This document assumes that the implimentor:
- has privileges to upload gateway-extensions to the CLoud Manager,
- knows how to develop apis on APIC,
- knows what DataPower Gateway is and has a gateway environment to execute the tasks detailed in this document.

For this example, we have an API, which contains a GatewayScript that reads a file on the gateway to pull port information from a lookup properties config file.
Use cases like this exist to ensure there is one externalized source for multiple code/scripts to reference on different environments to achieve portability of code and lessen having to change certain values in multiple apis many times when a change occurs.

The following gatewayscript uses the fs module to read the file local:///lookup_files/lookup_file_config.json  
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

