# Compliance task library

**Contents of this repository are intended for internal Microsoft use.**

This repository contains Azure DevOPS YAML template for the compliance tasks needed for release products.
The step templates can be included in the repository using [multi-checkout](https://docs.microsoft.com/en-us/azure/devops/pipelines/repos/multi-repo-checkout?view=azure-devops).

The following sample shows how the templates can be included in your release YAML.

1. Create a repository resource and a service connection to connect to this repository.

   ```yaml
   resources:
     repositories:
     - repository: ComplianceRepo
       type: github
       endpoint: ComplianceGHRepo
       name: PowerShell/compliance
   ```

1. In the compliance stage, checkout `self` repo and the `compliance` repo.

    ```yaml
    - stage: compliance
      displayName: Compliance
      dependsOn: Build
      jobs:
      - job: Compliance_Job
          pool:
            name: Package ES Standard Build
          steps:
          - checkout: self
          - checkout: ComplianceRepo
    ```

1. Pick one of the three composed templates,

    - `assembly-module-compliance.yml` - for running compliance for projects generating an assembly.
    - `script-module-compliance.yml` - for running compliance for projects generating a script module.
    - `ci-compliance.yml` - for running compliance as part of CI builds.

1. Call the template from this repo in your yaml file and specify the values for the parameters.

    ```yaml
    - template: assembly-module-compliance.yml@ComplianceRepo
        parameters:
            # binskim
            AnalyzeTarget: '$(Pipeline.Workspace)/*.dll'
            AnalyzeSymPath: 'SRV*'
            # component-governance
            sourceScanPath: '$(Build.SourcesDirectory)'
            # credscan
            suppressionsFile: ''
            # TermCheck
            optionsRulesDBPath: ''
            optionsFTPath: ''
            # tsa-upload
            codeBaseName: 'PSPager_202007'
            # selections
            APIScan: false # set to false when not using Windows APIs.
    ```

## ESRP Signing Template Overview

** Requires on-boarding, see the wiki in the internal PowerShell Maintainers teams channel **

Make sure to create the variable group named `ESRP` and make it available to the pipeline.
Details can be found in the PowerShell Maintainers teams channel's Wiki tab.

1. Call the template from this repo in your yaml file and specify the values for the parameters.

```yaml
- template: EsrpSign.yml@ComplianceRepo
    parameters:
        # the folder which contains the binaries to sign
        buildOutputPath: $(signSrcPath)
        # the location to put the signed output.
        # Note, All files in "buildOutputPath" are copied to "signOutputPath" not just ones matching the "pattern".
        signOutputPath: $(signOutPath)
        # the certificate ID to use
        certificateId: "CP-230012"
        # The file pattern to use
        # If not using minimatch: comma separated, with * supported
        # If using minimatch: newline separated, with !, **, and * supported.
        # See link in the useMinimatch comments.
        pattern: '*.dll,*.psd1,*.psm1,*.ps1xml,*.mof'
        # decides if the task should use minimatch for the pattern matching.
        # https://github.com/isaacs/minimatch#features
        useMinimatch: false
        # If "true", use a custom JSON string for ESRP signing. Defaults to "false".
        useCustomEsrpJson: false
        # If "true", ESRP will automatically verify your files are signed properly (eg signtool /verify).
        # Only supported for authenticode & nuget signing.  
        # Defaults to "false".
        verifySignature: false
        # If "true", ESRP will page hash sign your files.
        # Only supported for authenticode signing.
        # Defaults to "true".
        pageHash: true
        # If "true", ESRP will be called to sign your files.
        # If "false", ESRP is not called, files will not be signed.
        # If "auto", ESRP is called if and only certain build conditions are met.
        # Defaults to "auto".
        shouldSign: 'auto'
        # If "true", your files are always copied to signOutputPath even if ESRP is not called to sign the files.
        # Useful to set to true if you want your build pipeline to behave as close to 'normal' as possible when testing and not signing.
        # Defaults to "false".
        alwaysCopy: false
        # The name of the Azure DevOps Service connection you configured for ESRP Signing.
        # Defaults to "pwshSigning".
        signingService: 'pwshSigning'

```

### ESRP Authenticode minimatch example

This example signs `dll` and `psm1` files recursively and `psd1` files in the root of the `buildOutputPath`, using minimatch.

For full features see:  https://github.com/isaacs/minimatch#features

```yaml
  - template: EsrpSign.yml@ComplianceRepo
      parameters:
        buildOutputPath: $(signSrcPath)
        signOutputPath: $(signOutPath)
        certificateId: "CP-230012"
        pattern: |
          **\*.dll
          *.psd1
          **\*.psm1
        useMinimatch: true
```

### ESRP RPM example

This example signs `dll` `psd1` and `psm1` files recursively, using minimatch.

```yaml
  - template: EsrpSign.yml@ComplianceRepo
      parameters:
        buildOutputPath: $(signSrcPath)
        signOutputPath: $(signOutPath)
        # this is the cert for RPM signing
        certificateId: "CP-450779-Pgp"
        # this is the pattern for RPM signing
        pattern: |
            **\*.rpm
        useMinimatch: true
```


### ESRP NuPkg example

This example signs `dll` `psd1` and `psm1` files recursively, using minimatch.

```yaml
  - template: EsrpSign.yml@ComplianceRepo
      parameters:
        buildOutputPath: $(signSrcPath)
        signOutputPath: $(signOutPath)
        # this is the cert for NuPkg signing
        certificateId: "CP-401405"
        # this is the pattern for NuPkg signing
        pattern: |
            **\*.nupkg
        useMinimatch: true
```

### ESRP macOS example

This example signs `pkg` files recursively, using minimatch.

```yaml
  - template: EsrpSign.yml@ComplianceRepo
      parameters:
        buildOutputPath: $(signSrcPath)
        signOutputPath: $(signOutPath)
        # this is the cert for macOS signing
        certificateId: "CP-401337-Apple"
        # this is the pattern for pkg signing
        pattern: |
            **\*.pkg
        useMinimatch: true
```

### ESRP custom signing JSON example
1. Set the build variable ESRP_TEMPLATE_CUSTOM_JSON to your desired ESRP JSON string.
2. Call EsrpSign.yml@ComplianceRepo with certificateId: "" and useCustomEsrpJson: true.

```yaml
- pwsh: |
    [string] $SigningServer = '$(SigningServer)'
    Write-Verbose "SigningServer - $SigningServer" -Verbose

    $esrpParameters = @{
      OpusName   = "Microsoft"
      OpusInfo   = "http://www.microsoft.com"
      FileDigest = "/fd sha256"
      TimeStamp  = "/tr ""$SigningServer"" /td sha256"
    }

    $esrp = @(
      @{
        KeyCode = "Dynamic"
        CertTemplateName = "WINMSAPP1ST"
        CertSubjectName = "CN=Microsoft Corporation, O=Microsoft Corporation, L=Redmond, S=Washington, C=US"
        operationCode = "SigntoolSign"
        Parameters = $esrpParameters
        ToolName = "sign"
        ToolVersion = "1.0"},
      @{
        KeyCode = "Dynamic"
        CertTemplateName = "WINMSAPP1ST"
        CertSubjectName = "CN=Microsoft Corporation, O=Microsoft Corporation, L=Redmond, S=Washington, C=US"
        OperationCode = "SigntoolVerify"
        ToolName = "sign"
        ToolVersion = "1.0"
      }
    )

    $vstsCommandString = "vso[task.setvariable variable=ESRP_TEMPLATE_CUSTOM_JSON][$($esrp | ConvertTo-Json -Compress)]"
    Write-Verbose -Message ("sending " + $vstsCommandString) -Verbose
    Write-Host "##$vstsCommandString"
  displayName: Generate app signing JSON

- template: EsrpSign.yml@ComplianceRepo
    parameters:
        buildOutputPath: $(signSrcPath)
        signOutputPath: $(signOutPath)
        # Explicitly set to "" for custom string
        certificateId: ""
        pattern: '*.msix'
        useMinimatch: false
        # Use ESRP_TEMPLATE_CUSTOM_JSON as custom string
        useCustomEsrpJson: true
```

### ESRP Custom Signing Service Connection Example

This example uses a custom signing (Azure DevOps) service connection name.

```yaml
  - template: EsrpSign.yml@ComplianceRepo
      parameters:
        buildOutputPath: $(signSrcPath)
        signOutputPath: $(signOutPath)
        certificateId: "CP-230012"
        pattern: |
          **\*.dll
        # The name of the Azure DevOps Service connection you configured for ESRP Signing.
        # Defaults to "pwshSigning".
        signingService: 'FactoryOrchestratorSigning'

```

## ESRP Malware Scanning Template Overview

** Requires on-boarding, see the wiki in the internal PowerShell Maintainers teams channel **

Details can be found in the PowerShell Maintainers teams channel's Wiki tab.

This should be use in multi-Job builds when you are uploading unsigned binaries.
Files are automatically scanned on signing,
scanning on each upload will allow us to detect when any malware was introduced.

This should be use in multi-Job builds when you are uploading unsigned binaries.
Files are automatically scanned on signing,
scanning on each upload will allow us to detect when any malware was introduced.

1. Call the template from this repo in your yaml file and specify the values for the parameters.

```yaml

  - template: EsrpScan.yml@ComplianceRepo
    parameters:
        # the path with the files to scan
        scanPath: $(System.ArtifactsDirectory)
        # the minimatch pattern to find the files
        # https://github.com/isaacs/minimatch#features
        pattern: |
          **\*.rpm
          **\*.deb
          **\*.tar.gz
        # The name of the Azure DevOps Service connection you configured for ESRP Malware Scanning.
        # Defaults to "pwshSigning".
        scanningService: 'pwshEsrpScanning'
```

### ESRP Scanning Custom Service Example

This example uses a custom ESRP malware scanning (Azure DevOps) service name.

```yaml
  - template: EsrpSign.yml@ComplianceRepo
      parameters:
        buildOutputPath: $(signSrcPath)
        signOutputPath: $(signOutPath)
        certificateId: "CP-230012"
        pattern: |
          **\*.dll
        scanningService: 'FactoryOrchestratorScanning'

```