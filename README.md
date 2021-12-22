# Amazon-Connect-Copy User Guide

The Amazon-Connect-Copy script copies components from the source Amazon Connect instance
to the target instance safely, fixing all internal references.

You may use Amazon-Connect-Copy to deploy an Amazon Connect instance across environments
(AWS accounts or regions), or to save a backup copy of the instance for restoration
when required, reducing an hours-long error-prone manual job to few minutes of
automated and reliable processing.

Ids and Arns of components copied from the source instance will be re-mapped to
their corresponding components in the new instance, including:

- Instance (pre-existing)
- Lambda functions (pre-deployed)
- Prompts (pre-uploaded)
- Hours of operations
- Queues
- Routing profiles
- Contact flow modules
- Contact flows

These components are not modified by Amazon-Connect-Copy (to avoid any impact on
other contact centres that may happen to be using the same target instance):

- Users (agents) related settings
- Security profiles
- Phone numbers
  - Inbound Contact flow/IVR mappings
  - Outbound caller ID number for queues
- Quick connects
- Settings for existing queues
  - Note: Settings for new queues will still be copied

Considering the target instance may accommodate multiple contact centres,
Amazon-Connect-Copy does not remove any target instance components which are
not found in the source instance. If there are multiple contact centres sharing
the same Amazon Connect instance, it is a good practice to prefix contact centre
specific components with their individual Contact Centre Codes (CCC).

## Installation

- Install [AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html).
  - Recommend installing the latest version of AWS CLI.
- Install [jq](https://stedolan.github.io/jq/) (if not already installed on your platform).
- Copy `bin/*` to your Shell search path (e.g., `cp bin/* /usr/local/bin/`).

## Example

**Please replace names in the example with names specific to your use case.**

```
connect_save source-connect source-profile CCC
connect_save target-connect target-profile CCC
connect_diff source-connect target-connect target-helper source- target-
# Dry run
connect_copy -d target-helper
# Real run
connect_copy target-helper
```

In this example:
- We copy from instance `source-connect` to instance `target-connect`.
- Credentials for the source and target instances are in AWS profiles
  `source-profile` and `target-profile` respectively.
- You may use the same profile for `source-profile` and `target-profile`,
  as long as that profile allows access to both the source and the target instances
  (typically when they are in the same AWS account).
- Differences of the two instances (including profile specifications)
  will be saved in directory `target-helper`.
- All Lambda functions are assumed deployed with function names prefixed by
  `source-` for the source instance and `target-` for the target instance.
  (These two arguments can be omitted if the Lambda functions used in the source
  and target instances bear the same name.)
- Only contact flows and modules with names prefixed by the `CCC`
  Contact Centre Code will be copied to the target instance.

## Copying process

Note: All names in Amazon Connect are case sensitive.

### Pre-steps

- Make sure no one else is making changes to either the source or the
  target instances, or any Lambda functions they invoke.
- Deploy all Lambda functions required by the target instance.
- Upload all required prompts to the target instance.
  - The prompt names need to be *exactly* the same as their counterparts
    in the source instance.
- For an incremental instance change, if there are contact flow or module name changes,
  before the copying please manually change the corresponding flow or module names in
  the *target* instance. Otherwise, contact flows and modules with new names will
  be created in the target instance, and those with old names will be left untouched.

### Copying

- Set up [named profiles](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-profiles.html) for AWS CLI access to Amazon Connect.
  - `<source_profile>` for the source instance
  - `<target_profile>` for the target instance
  - This step is optional if your default profile already has access to both
    the source and the target instances. If not sure, skip this step for now.
    You only need to set up the profiles if `connect_save` fail due to a permission error.
- `cd` to an empty working directory (e.g., `md <dir>; cd <dir>`).
- Optionally, run `connect_save` with no arguments to show the help message:
  ```
  Usage: connect_save [-?fs] [-G ignore_prefix] instance_alias [aws_profile] [contact_flow_prefix]
      Retrieve resources from an Amazon Connect instance into plain files

      instance_alias       Alias of the Connect instance (or path to the directory to save, with the alias being the basename)
      profile              AWS Profile to use
      contact_flow_prefix  Prefix of Contact Flows and Modules of interest (all others will be ignored)
      -f                   Force removal of existing instance_alias directory
      -s                   Skip unpublished contact flow modules and contact flows with an error, instead of failing
      -G ignore_prefix     Ignore hours, queues, routing profiles, flows or modules with names prefixed with ignore_prefix
      -?                   Help
  ```
  - `<instance_alias>` can be a directory path.
  - `<instance_alias>.log` will be produced by `connect_save`.
- Run `connect_save <source_instance_alias> <source_profile> <contact_flow_prefix>` .
- Run `connect_save <target_instance_alias> <target_profile> <contact_flow_prefix>` .
- Optionally, run `connect_diff` with no arguments to show the help message:
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
- Run `connect_diff <source_instance_alias> <target_instance_alias> <helper> <source_lambda_prefix> <target_lambda_prefix>` .
- Optionally, check under the helper directory `<helper>` to find the four helper files:
  - `helper.new` - components to create
     (those found in the source but not in the target)
  - `helper.old` - components to update
    (those found in the source and also in the target)
  - `helper.sed` - SED script to fix references
    (so target components will not refer to any components in the source)
  - `helper.var` - variables of the two instances
    (instance A is the source, and instance B is the target)
- Optionally, run `connect_copy` with no arguments to show the help message:
  ```
  Usage: connect_copy [-?d] helper
      Copy Amazon Connect instance A to instance B safely, based on the
      connect_list and connect_diff results, under the helper directory
      creating new components in helper.new, updating old components in helper.old,
      and updating references defined in helper.sed.

      helper        Name of the helper directory
      -d            Dry run - Run through the script but not updating the target instance
      -?            Help
  ```
- Optionally, verify the helper by dry-running `connect_copy -d <helper>` .
  - Check if the proposed changes are as what you would expect from the output.
  - AWS CLI commands to execute can be found in `<helper>.log` for your reference.
  - Please do not run the log file as an executable. Run `connect_copy`
    without `-d` (dry-run) to perform the actual copying.
- Run `connect_copy <helper>` .
  - Verify if the target instance contains all source instance components
    of the latest version, with all internal references and Lambda invocations
    properly adjusted.

### Post-steps

- Login to Amazon Connect target instance.
- Open Phone numbers.
  - Check Contact flow/IVR of all phone numbers.
  - If required, re-map phone numbers to the new Inbound Contact flows/IVR.
- Open Queues.
  - For new outbound queues created, set these if required:
    - Outbound caller ID name
    - Outbound caller ID number
    - Outbound whisper flow

## Backup an Amazon Connect instance using Amazon-Connect-Copy

You may restore an Amazon Connect instance from a previous backup copy saved by `connect_save`.

Example:
- Save a backup copy of the Amazon Connect instance.
  ```
  connect_save <backup_dir>/<connect_instance_alias> <profile>
  ```
- Restore the same instance from the backup copy.
  - Save the current copy (the one to be restored).
    ```
    connect_save <working_dir>/<connect_instance_alias> <profile>
    ```
  - Diff the current copy with the backup copy.
    ```
    connect_diff <backup_dir>/<connect_instance_alias> <working_dir>/<connect_instance_alias> <helper_dir>
    ```
  - Optionally, dry run to verify restoration changes.
    ```
    connect_copy -d <helper_dir>
    ```
  - Restore the instance.
    ```
    connect_copy <helper_dir>
    ```

## Useful Tips

- This script has been tested with AWS CLI 2.4.1, which supports the latest
  Amazon Connect features, including Contact Flow Modules.
  (Even your instances may not be using all latest Amazon Connect features,
  the script will check them and therefore require the latest AWS CLI.)
- Make sure both the source instance and the target instance remain unaltered
  by anyone during the entire copying process (save, diff and copy).
- `connect_diff` only creates the helper directory and will not change anything
  in the source and the target instance directories.
- `connect_copy` will change files in the helper directory, and when in
  non-dry-run mode, will change the target instance directory as well.
  i.e., `connect_copy` is not idempotent to the target and helper directory.
- DO NOT reuse the target instance directory and the helper directory.
  Remove these two directories after copying.
  - If you want to keep a backup of the target instance after copying,
    run `connect_save` again on the target instance.
- `connect_diff` and `connect_copy` do not change the source instance directory,
  so the source can serve as a backup or be used to copy to multiple target instances.
- If relative paths are specified in instance aliases, make sure you are running
  `connect_diff` and `connect_copy` from the same directory, so that `connect_copy`
  will resolve the relative paths correctly.
- The Amazon-Connect-Copy script does not affect any instance-specific settings outside of
  the Amazon Connect console, such as
  [Amazon Connect service quotas](https://docs.aws.amazon.com/connect/latest/adminguide/amazon-connect-service-limits.html).
