---
title: Upgrade NVOS/CPLD Firmware
author: NVIDIA
weight: 850
toc: 4
---

Use the REST API to upgrade NVOS or CPLD Firmware on your switches. First, create a switch profile, then upload the NVOS/CPLD image and track the progress of the upgrade.

## Requirements

- Download the NVOS/CPLD image from the {{<exlink url="https://enterprise-support.nvidia.com/s/" text="NVIDIA Enterprise Support Portal">}}.
- Only the admin or read-write user (`rw-user`) can perform the steps in this section.

## Upgrade NVOS/CPLD Firmware

You can upgrade NVOS/CPLD Firmware at either the switch-level or domain-level. Switch-level operations are prioritized before domain-level operations.

{{<tabs "TabID129" >}}
{{<tab "Switch-level upgrade"  >}}

1. Authenticate your credentials by {{<link title="NVLink Bringup/#switch-profile-endpoints" text="creating a switch profile">}}. Make a POST request to the `/v1/switch-profiles` endpoint that contains your username and password. Copy the `ProfileID` from the response body.

2. Upload the NVOS/CPLD image by making a POST request to the `/v1/images` endpoint with a corresponding `ImageType` field. After the image is successfully uploaded, the response body returns an `ImageID`. 

`ImageType` must be one of the following: `nvos` or `cpld`, case-sensitive.

3. Make a POST request to the `/v1/upgrade-switch` endpoint with a corresponding `UpgradeType` field; select **switch-based upgrade** and include the `ProfileID` and `ImageID` from the previous steps, in addition to the IP addresses of the switches you want to upgrade.

`UpgradeType` must be one of the following: `nvos` or `cpld`, case-sensitive.
`ImageType` of the image must match the `UpgradeType`.

4. If all initial validations succeed, the API returns an `HTTP 202 Accepted` response with a JSON body containing an operation ID. You can make a GET request to the `/v1/operations/` endpoint to track the progress of the upgrade.

{{</tab>}}
{{<tab "Domain-level upgrade" >}}

1. Authenticate your credentials by {{<link title="NVLink Bringup/#switch-profile-endpoints" text="creating a switch profile">}}. Make a POST request to the `/v1/switch-profiles` endpoint that contains your username and password. Copy the `ProfileID` from the response body.

2. Upload the NVOS/CPLD image by making a POST request to the `/v1/images` endpoint with a corresponding `ImageType` field. After the image is successfully uploaded, the response body returns an `ImageID`.

`ImageType` must be one of the following: `nvos` or `cpld`, case-sensitive.

3. Make a PATCH request to the `/v1/domains/{id}` endpoint to create an association between the profile ID from the first step and a given domain. The response body returns a `DomainID`.

4. Make a POST request to the `/v1/upgrade-switch` endpoint with a corresponding `UpgradeType` field; select **domain-based upgrade** and include the `DomainID` and `ImageID` from the previous steps.

`UpgradeType` must be one of the following: `nvos` or `cpld`, case-sensitive.
`ImageType` of the image must match the `UpgradeType`.

5. If all initial validations succeed, the API returns an `HTTP 202 Accepted` response with a JSON body containing an operation ID. You can make a GET request to the `/v1/operations/` endpoint to track the progress of the upgrade.

{{</tab >}}
{{</tabs>}}
