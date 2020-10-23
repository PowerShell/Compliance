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
       name: adityapatwardhan/compliance
   ```

1. In the compliance stage, checkout `self` repo and the `compliance` repo.

    ```yaml
    - stage: compliance
      displayName: Compliance
      dependsOn: Build
      jobs:
      - job: Compliance_Job
          pool:
          name: Package ES CodeHub Lab E
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

## ESRP Template Example

** Requires on-boarding, see the wiki in the internal PowerShell Maintainers teams channel **

1. Call the template from this repo in your yaml file and specify the values for the parameters.

    ```yaml
    - template: EsrpSign@ComplianceRepo
        parameters:
           # the folder which contains the binaries to sign
           buildOutputPath: $(signSrcPath)
           # the location to put the signed output
           signOutputPath: $(signOutPath)
           # the certificate ID to use
           certificateId: "CP-230012"
           # the file pattern to use, comma separated
           pattern: '*.dll,*.psd1,*.psm1,*.ps1xml,*.mof'
    ```



