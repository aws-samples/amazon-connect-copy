# Amazon-Connect User Guide

Repository - "amazon-connect-copy-dev" - Branch dev-mehulaws
===========================================================
Please refer to INSTRUCTIONS for implementation details

The Amazon-Connect Migration script copies components from the source Amazon Connect instance
to the target instance safely, fixing all internal references.

You may use this script to deploy an Amazon Connect instance across environments
(AWS accounts or regions), or to save a backup copy of the instance for restoration
when required, reducing an hours-long error-prone manual job to a few minutes of
automated and reliable processing.

Ids and Arns of components copied from the source instance will be re-mapped to
their corresponding components in the new instance, including:

- Instance (pre-existing)
- Lambda functions (pre-deployed)
- Lex bots (Classic) (pre-deployed)
- Prompts (pre-uploaded)
- Hours of operations
- Queues
- Routing profiles
- Contact flow modules
- Contact flows
- Quick Connects

The following components are not copied by Amazon-Connect-Copy (to avoid any impact on
other contact centres that may happen to be using the same target instance):

- Users (agents) related settings, statuses and the hierarchy
- Security profiles
- Phone numbers
  - Inbound Contact flow/IVR mappings
  - Outbound caller ID number for queues
- Settings for existing queues
  - Note: Settings for new queues will still be copied
- Historical metrics and reports
- Contact Trace Records (CTRs)
- Custom vocabularies
- Rules for Contact lens and Third-party integration

Considering the target instance may accommodate multiple contact centres,
the script does not remove any target instance components which are
not found in the source instance. If there are multiple contact centres sharing
the same Amazon Connect instance, it is a good practice to prefix contact centre
specific components with their individual Contact Centre Codes (CCCs).

A note to developers: We will accept PRs on the "development" branch only. Thank you.

## Installation

- Install [AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html).
  - Recommend installing the latest version of AWS CLI.
- Install [jq](https://stedolan.github.io/jq/) (if not already installed on your platform).
- Copy `bin/ac_migrate.zip to your your environment and unzip it

Refer to INSTRUCTIONS for additional details
