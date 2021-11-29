# Connect-Copy Notes

The Connect-Copy script copies components from the source Amazon Connect instance to the target instance safely,
fixing all internal references.

You may use Connect-Copy to deploy an Amazon Connect instance across environments,
or to save a backup copy of the instance for restoration when required,
reducing an hours-long error-prone manual job to few minutes of automated and reliable processing.

Ids and Arns of components copied from the source instance will be re-mapped to their corresponding components
in the new instance, including:

- Instance (pre-existing)
- Lambda functions (pre-deployed)
- Prompts (pre-uploaded)
- Hours of operations
- Queues
- Routing profiles
- Contact flow modules
- Contact flows

These components are not modified by Connect-Copy
(to avoid any impact on other contact centres that may happen to be using the same target instance):

- Users (agents) related settings
- Security profiles
- Phone numbers
  - Inbound Contact flow/IVR mappings
  - Outbound caller ID number for queues
- Quick connects
- Settings for existing queues
  - Note: Settings for new queues will still be copied

Considering the target instance may accommodate multiple contact centres,
Connect-Copy does not remove any target instance components which are not found in the source instance.
If there are multiple contact centres sharing the same Amazon Connect instance, it is a good practice
to prefix contact centre specific components with their individual Contact Centre Codes (CCC).

## Installation

- Install [AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html)
  - Recommend installing the latest version of AWS CLI
- Install [jq](https://stedolan.github.io/jq/) (if not already installed on your platform)
- Copy bin/* to your shell search path (e.g., /usr/local/bin)

## Example

In this example:
- We copy from instance `source-connect` to instance `target-connect`.
- Only contact flows and modules with names prefixed by the `CCC` Contact Centre Code will be copied.
- Credentials for the source and target instances are in AWS profiles
  `source-profile` and `target-profile` respectively.
- You may use the same profile for `source-profile` and `target-profile`,
  as long as that profile allows access to both the source and the target instances
  (typically when they are in the same AWS account).
- All Lambda functions are assumed deployed with function names prefixed by `source-` for
  the source instance and `target-` for the target instance. These two arguments can be omitted
  if the same set of Lambda functions are used for both instances.
- Resources (queues, routing profiles and contact flows/modules) with names prefixed by `zz` will be ignored
  (by the `-G` option).
- Differences of the two instances (including profile specifications) will be saved in directory `target-helper`.
- Only contact flows and modules with names prefixed by `CCC` will be copied to the target instance.

**Please replace names in the example with names specific to your use case.**

```
bin/connect_save -f -G zz source-connect source-profile
bin/connect_save -f -G zz target-connect target-profile
bin/connect_diff -f source-connect target-connect target-helper source- target-
# Dry run
bin/connect_copy -g CCC -d target-helper
# Real run
bin/connect_copy -g CCC target-helper
```

## Copying process

Note: All names in Amazon Connect are case sensitive.

### Pre-steps

- Make sure no one else is making changes to either the source or the target instances,
  or any Lambda functions they invoke.
- Deploy all Lambda functions required by the target instance.
- Upload all required prompts to the target instance.
  - The prompt names need to be exactly the same as their counterparts in the source instance.
- For an incremental instance change, if there are contact flow/module name changes,
  before the copying please manually change the corresponding flow/module names in the *target* instance.
  Otherwise, contact flows and modules with new names will be created in the target instance,
  and those with old names will be left untouched.

### Copying

- Set up [named profiles](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-profiles.html) for AWS CLI access to Amazon Connect.
  - `<source_profile>` for the source instance
  - `<target_profile>` for the target instance
  - This step is required only if `bin/connect_save` failed due to a permission error.
- `cd` to an empty working directory (e.g., `md <dir>; cd <dir>`).
- Unpack the Connect-Copy scripts into the clean working directory.
  - Alternatively, you can copy the scripts under `bin/` to your PATH so you don't need to include `bin/` in each command.
- Optionally, run `bin/connect_save` with no arguments to show the help message:
  ```
  Usage: connect_save [-?f] [-G ignore_prefix] instance_alias [aws_profile] [contact_flow_prefix]
      Retrieve resources from an Amazon Connect instance into plain files

      instance_alias       Alias of the Connect instance (or path to the directory to save, with the alias being the basename)
      profile              AWS Profile to use
      contact_flow_prefix  Prefix of Contact Flows and Modules of interest (all others will be ignored)
      -f                   Force removal of existing instance_alias directory
      -G ignore_prefix     Ignore queues, routing profiles or flows/modules with names prefixed by ignore_prefix
      -?                   Help
  ```
- Run `bin/connect_save <source_instance_alias> <source_profile> <contact_flow_prefix>`
- Run `bin/connect_save <target_instance_alias> <target_profile> <contact_flow_prefix>`
- Optionally, run `bin/connect_diff` with no arguments to show the help message:
  ```
  Usage: connect_diff [-?f] instance_alias_a instance_alias_b helper [lambda_prefix_a] [lambda_prefix_b]
      Based on connect_list result on Amazon Connect instance A and B,
      find the differences and produce helper files to safely copy resources from A to B.

      instance_alias_a  Alias of the Connect instance A
      instance_alias_b  Alias of the Connect instance B
                        (Aliases can be a path to the directory where the instance was saved using connect_save.)
      helper            Name of the helper directory
      lambda_prefix_a   Lambda function name prefix in instance A to be replaced with <lambda_prefix_b>
      lambda_prefix b   Lambda function name prefix in instance B replacing <lambda_prefix_a>
      -f                Force removal of existing helper directory
      -?                Help

      Note: This script create files in the helper directory without changing any instance resource files.
  ```
- Run `connect_diff <source_instance_alias> <target_instance_alias> <helper>`
- Check under the helper directory `<helper>` to find the four helper files:
  - `helper.new` - components to create (those found in the source but not in the target)
  - `helper.old` - components to update (those found in the source and also in the target)
  - `helper.sed` - SED script to fix references (so target components will not refer to any components in the source)
  - `helper.var` - variables of the two instances (instance A is the source, and instance B is the target)
- Optionally, run `bin/connect_copy` with no arguments to show the help message:
  ```
  Usage: connect_copy [-?d] [-g cf_prefix] helper
      Copy Amazon Connect instance A to instance B safely, based on the
      connect_list and connect_diff results, under the helper directory
      creating new components in helper.new, updating old components in helper.old,
      and updating references defined in helper.sed.

      helper        Name of the helper directory
      -d            Dry run - Run through the script but not updating the target instance
      -g cf_prefix  Only copy contact flows/modules with their name prefixed by cf_prefix
      -?            Help
  ```
- Optionally, verify the helper by dry-running `connect_copy -g <contact_flow_prefix> -d <helper>`
  - Check if the proposed changes are as what you would expect.
- Run `connect_copy -g <contact_flow_prefix> <helper>`
  - Verify if the target instance contains all source instance components of the latest version,
    with all internal references and Lambda invocations properly adjusted.

### Post-steps

- Login to Amazon Connect target instance
- Open Phone numbers
  - Check Contact flow/IVR of all phone numbers
  - If required, re-map phone numbers to the new Inbound Contact flows/IVR
- Open Queues
  - For new outbound queues created, set these if required:
    - Outbound caller ID name
    - Outbound caller ID number
    - Outbound whisper flow

## Backup an Amazon Connect instance using Connect-Copy

You may restore an Amazon Connect instance from a previous backup copy saved by `connect_save`.

Example:
- Save a backup copy of the Amazon Connect instance
  ```
  bin/connect_save -f <backup_dir>/<connect_instance_alias> <profile>
  ```
- Restore the same instance from the backup copy
  - Save the current copy (the one to be restored)
    ```
    bin/connect_save -f <working_dir>/<connect_instance_alias> <profile>
    ```
  - Diff the current copy with the backup copy
    ```
    bin/connect_diff -f <backup_dir>/<connect_instance_alias> <working_dir>/<connect_instance_alias> <helper_dir>
    ```
  - Optionally, dry run to verify restoration changes
    ```
    bin/connect_copy -d <helper_dir>
    ```
  - Restore the instance
    ```
    bin/connect_copy <helper_dir>
    ```

## Useful Tips

- Make sure both the source instance and the target instance remain unaltered by anyone during the entire copying process.
- `connect_diff` only creates the helper directory and will not change anything
  in the source and the target instance directories.
- `connect_copy` will change files in both the target instance directory and the helper directory.
- DO NOT reuse the target instance directory and the helper directory. Remove these two directories after copying.
  - If you want to keep a backup of the target instance after copying, run `connect_save` again.
- `connect_diff` and `connect_copy` do not change the source instance directory, so it can be served as a backup.
- If relative paths are specified in instance aliases, make sure you are running `connect_diff` and `connect_copy`
  from the same working directory, so that `connect_copy` will resolve the relative paths correctly.
