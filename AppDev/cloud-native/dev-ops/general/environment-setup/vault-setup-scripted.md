![](../../../../../common/images/customer.logo2.png)

# OCI Vault Creation

## Introduction

One of the ways you can pass information into a OCI DevOps build pipeline is to store it in a secret in an OCI vault. To do this of course we need to create a vault, and a master key to encrypt secrets held within the vault.

### Objectives

Using the script `vault-setup.sh` to create the vault used to hold the secrets that contain the configuration information.

### Prerequisites

You have downloaded the scripts into your OCI CLoud shell instance, have run the `initials-setup.sh` and the `compartment-setup.sh` scripts

## Task 1: Creating the vault

In this module we will use a script to setup the vault as that is not a core part of understanding the devops process, however there are of course multiple ways you could create the vault and master signing key, for example using the OCI BUI or terraform.

  1. Open the OCI Cloud shell
  
  2. Make sure you are in the home directory
  
  - `cd $HOME/helidon-kubernetes/setup/devops-labs`
  
  3. Run the script
  
  - `bash vault-setup.sh`

```
No reuse information for vault
Operating in compartment CTDOKE
Do you want to use tgDevOps as the name of the vault to create or re-use in CTDOKE?y
```

  4. When asked if you want to use the details name type `y` - unless you want to chose a different name, in which case type `n` and when prompted enter your preferred name for the vault. Remember you will need to substitute any new name later in the lab. The script will wait for the vault to be created before proceeding, this may take a few minutes, so don't worry if the waiting stage does not continue immediately.
  
```
Checking for active vault tgDevOps in compartment CTDOKE
Vault named tgDevOps doesn't exist, creating it, there may be a short delay
Action completed. Waiting until the resource has entered state: ('ACTIVE',)
Vault being created using OCID ocid1.vault.oc1.uk-london-1.cfq4gscbaadig.abwgiljtmtc3f34texllqsk7gy3ynxi3s3mcjy7pxv4sujeofm5zzufinxhq
Setting up for Vault master key
Getting vault endpoint for vault OCID ocid1.vault.oc1.uk-london-1.cfq4gscbaadig.abwgiljtmtc3f34texllqsk7gy3ynxi3s3mcjy7pxv4sujeofm5zzufinxhq
Checking for existing key named tgKey in endpoint https://cfq4gscbaadig-management.kms.uk-london-1.oraclecloud.com in compartment OCID ocid1.compartment.oc1..aaaaaaaaan2yrsn3lcsaectitem7swo523ysfszii3e57btglyymcooycu3a
No existing key with name tgKey, creating it
echo Vault master key created with OCID ocid1.key.oc1.uk-london-1.cfq4gscbaadig.abwgiljsc2fjle4rxi5cwgvvnjffn6ypospo2myk4trm7tplaepjjsutiv2a
```

Once the script has completed the vault will have been created with the specified name and a master signing key will be created named `<YOUR_INITIALS>Key`

## End of the Module, what's next ?

Congratulations, you're now ready to use your vault to hold secret configuration information, The next step is to configure your environment for ssh access.

## Acknowledgements

* **Author** - Tim Graves, Cloud Native Solutions Architect, EMEA OCI Centre of Excellence
* **Last Updated By** - Tim Graves, December 2021
  