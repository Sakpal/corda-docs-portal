---
date: '2023-08-10'
version: 'Corda 5.1'
title: "net.corda.v5.membership"
menu:
  corda51:
    identifier: corda51-api-membership
    parent: corda51-api
    weight: 6000
section_menu: corda51
---
# net.corda.v5.membership
The `corda-membership` module defines interfaces that provide information about a {{< tooltip >}}member{{< /tooltip >}} and a {{< tooltip >}}membership group{{< /tooltip >}}. The interfaces in this module should not be implemented by {{< tooltip >}}CorDapp{{< /tooltip >}} Developers. Instead, instances can be retrieved through lookup services.

This module consists primarily of the following two root classes:
* [MemberInfo](#memberinfo)
* [GroupParameters](#groupparameters)

## `MemberInfo`
The `MemberInfo` interface exposes properties of a virtual node's membership. This includes the {{< tooltip >}}X.500{{< /tooltip >}} name, {{< tooltip >}}ledger keys{{< /tooltip >}}, and status. This information is a combination of information provided during network registration and metadata assigned to the member by the {{< tooltip >}}MGM{{< /tooltip >}}.

Information provided by the virtual node operator at time of registration is the content of the `MemberContext` and the information provided by the MGM is the source of the `MGMContext` content.

Instances of `MemberInfo` must be retrieved through a lookup API. `MemberLookup` from the <a href="application/membership.md">`corda-application` module</a>, is the lookup API which provides membership information to CorDapps. This can be used to look up the information of the member executing a {{< tooltip >}}flow{{< /tooltip >}} or other members available to transact with within the group.

The `MemberInfo` interface extends the `LayeredPropertyMap` interface, which means that membership information is key-value `String` pairs that are parsed and returned through properties. Generally, any properties required for use within a CorDapp are exposed through the `MemberInfo` interface. Other properties may only be relevant internally or at certain layers within the codebase so these are exposed through extension functions.


## `GroupParameters`

The `GroupParameters` interface is also a type of `LayerPropertyMap` which exposes properties of the group as distributed by the network manager (MGM). These properties define the parameters under which all members must operate during transactions.

The MGM can update the current group parameters using the MGM API. This update may only include changes to the minimum platform version and custom properties. To update the group parameters, the MGM must submit an updated version of the group parameters to the REST endpoint, which overwrites any previous properties submitted using the endpoint.

To view current group parameters:

{{< tabs >}}
{{% tab name="Bash"%}}

```shell
curl --insecure -u admin:admin -X GET $API_URL/members/$HOLDING_ID/group-parameters
```

{{% /tab %}}
{{% tab name="PowerShell" %}}

```shell
 Invoke-RestMethod -SkipCertificateCheck  -Headers @{Authorization=("Basic {0}" -f $AUTH_INFO)} -Uri "$API_URL/membership/$HOLDING_ID/group-parameters" | ConvertTo-Json -Depth 4
 ```

 To submit a group parameters update, use the MGM API endpoint as shown below. Keys of custom properties must have the prefix "ext.". For minimum platform version, use key "corda.minimum.platform.version".
{{< tabs >}}
{{% tab name="Bash"%}}

```shell
export GROUP_PARAMS_UPDATE='{"newGroupParameters":{"corda.minimum.platform.version": "50000", "ext.group.key.0": "value0", "ext.group.key.1": "value1"}}'
curl --insecure -u admin:admin -d "GROUP_PARAMS_UPDATE" $API_URL/mgm/$HOLDING_ID/group-parameters
```

{{% /tab %}}
{{% tab name="PowerShell" %}}

```shell
GROUP_PARAMS_UPDATE = @{
  'corda.minimum.platform.version' = "50000"
  'ext.group.key.0' = "value0"
  'ext.group.key.1' = "value1"
}
$GROUP_PARAMS_UPDATE_RESPONSE = Invoke-RestMethod -SkipCertificateCheck  -Headers @{Authorization=("Basic {0}" -f $AUTH_INFO)} -Method Post -Uri "$API_URL/mgm/$HOLDING_ID/group-parameters" -Body (ConvertTo-Json -Depth 4 @{
    newGroupParameters = $GROUP_PARAMS_UPDATE
})
$GROUP_PARAMS_UPDATE_RESPONSE.parameters
```

These submitted parameters are combined with notary information from the platform to construct the new group parameters, which are then signed by the MGM and distributed within the network.

{{< note >}}
Custom properties have a character limit of 128 for keys and 800 for values. A maximum of 100 such key-value pairs may be defined.
{{</ note >}}
