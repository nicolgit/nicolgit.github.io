---
title: Azure Open AI recipes for Azure API Management Service
date: 2025-01-01 10:00
tags: [Azure, networking, API manager, OpenAI ]
excerpt: "A collection of recipes for API Management for those who need to expose one or more instances of Azure OpenAI"

header:
  overlay_image: https://live.staticflickr.com/65535/54308248466_dc89a5f3d7_h.jpg
  caption: "Photo credit: [**nicola since 1972**](https://www.flickr.com/photos/15216811@N06/54308248466/)"
---

Azure OpenAI Service provides REST API access to OpenAI's powerful language models including o1, o1-mini, GPT-4o, GPT-4o mini, GPT-4 Turbo with Vision, GPT-4, GPT-3.5-Turbo, and Embeddings model series. These models can be easily adapted to your specific task including but not limited to content generation, summarization, image understanding, semantic search, and natural language to code translation.

Azure API Management, on the other hand, is a comprehensive API management platform that helps enterprises manage, secure, and monitor their APIs. It provides features such as API gateway, rate limiting, analytics, and developer portal, making it easier for businesses to expose their services to internal and external consumers.

By using Azure API Manager in front of Azure Open AI instances, enterprises can ensure better control, security, and visibility over their AI deployments. This setup allows for centralized management of API traffic, improved performance through caching and load balancing, and enhanced security with features like authentication and authorization.

Combining Azure Open AI with Azure API Manager enables enterprises to efficiently distribute and manage their AI capabilities while maintaining high standards of security and performance.

**In this post I show a collection of recipes for API Management that I have seen used by customers and partners who needed to expose one or more instances of Azure OpenAI.**

> Network configuration and integration with an Enterprise-scale landing zone (ESLZ) is out of scope. For a walkthrough that guides the integration of APIM and AOAI in a hub & spoke context, you can also refer to [my article](https://github.com/nicolgit/hub-and-spoke-playground/blob/main/scenarios/aoai.md) available as part of [the hub-and-spoke playground project](https://github.com/nicolgit/hub-and-spoke-playground).

The lab used for these walk-throughs consists of the following resources:

* Azure API Management service (developer SKU) `nicold-apim`
* 2 x Azure OpenAI service (S0 SKU) `apimaoai01` and `apimaoai02`

![architecture](../../assets/post/2025/apim-aoai/00-architecture.png)

this is the list of topic I will cover:

- [Add Azure Open AI as backend resource](#add-azure-open-ai-as-backend-resource)
- [Show `apim-001` endpoint as root API](#show-apim-001-endpoint-as-root-api)
- [Implement throttling](#implement-throttling)
- [Show in a Response header (_aoai-origin_) the host of the OpenAI API Called](#show-in-a-response-header-aoai-origin-the-host-of-the-openai-api-called)
- [Round robin calls between 2 instances of Open AI](#round-robin-calls-between-2-instances-of-open-ai)
- [Fallback on a second openAI instance for 30 secs. If the first send a 429 error (too many requests)](#fallback-on-a-second-openai-instance-for-30-secs-if-the-first-send-a-429-error-too-many-requests)
- [Generate a report of API Usage by access key](#generate-a-report-of-api-usage-by-access-key)
- [Generate a report of API Usage by source IP](#generate-a-report-of-api-usage-by-source-ip)
- [Generate a report of backend usage over time](#generate-a-report-of-backend-usage-over-time)
- [Generate a report of API requests by OpenAI deployment-id used](#generate-a-report-of-api-requests-by-openai-deployment-id-used)

# Add Azure Open AI as backend resource

Go to API Management services > `nicold-apim` > Backends > Add
* Name: `apim-001`
* Backend hosting type: Custom URL
* Runtime URL: https://nicold-aoai-001.openai.azure.com/openai
* Authorization credential
    * Headers
        * Name: `api-key`
        * Key: _your endpoint key_
* Click [create]

Go to API Management services > `nicold-apim` > Backends > Add
* Name: `apim-002`
* Backend hosting type: Custom URL
* Runtime URL: https://nicold-aoai-002.openai.azure.com/openai
* Authorization credential
    * Headers
        * Name: `api-key`
        * Key: _your endpoint key_
* Click [create]

Here is the result:
![backend resources added](../../assets/post/2025/apim-aoai/01-add-backend-resources.png)

# Show `apim-001` endpoint as root API

Go to Azure Portal > API Management Services > `nicold-apim` > API > Create from Azure Resources > Azure OpenAI Service
* Azure OpenAI instance: `nicold-aoai-001`
* API Version: `2024-02-01`
* Display name: `/`
* Name: `root-api`
* Click [review and create] and then [create]

This creates a frontend but ALSO a backend. To use the previously created backend, go to:

API Management Services > `nicold-apim` > APIs > All APIs > `/` > Design > Backend > Policies > Base

and change `backend-id="openai-root-openai-endpoint"` to `backend-id="apim-001"`, then save it.

To test this endpoint, go to: API Management Services > `nicold-apim` > APIs > All APIs > `/` > Test > `Creates a completion for the chat message`
* Deployment-id: `gpt4o-001`
* API-version: `2024-02-01` (same as used above)
* Request body: `{"messages": [{ "role": "system","content": "You are a helpful assistant."},{ "role": "user", "content": "Tell me a joke!"} ]}`
* Click: [SEND]

Here is also a PowerShell script to test this configuration:

```ps1
$openai = @{
   api_key     = "my-api-key"
   api_base    = "https://nicold-apim.azure-api.net/" # your endpoint
   api_version = '2024-02-01'
   name        = 'gpt4o-001' # custom name you chose for your deployment
}

$body = '{
  "messages": [
    { "role": "system","content": "You are a helpful assistant."},
    { "role": "user", "content": "Tell me a joke!"}
  ]}'

# Header for authentication
$headers = [ordered]@{
   'api-key' = $openai.api_key
}

# Send a request to generate an answer
$url = "$($openai.api_base)/deployments/$($openai.name)/chat/completions?api-version=$($openai.api_version)"

$response = Invoke-WebRequest -Uri $url -Headers $headers -Body $body -Method Post -ContentType 'application/json'

# Show response headers
$response.headers
$responseObj = ConvertFrom-Json $response.content

# Show response body
$responseObj.choices.message.content
```

# Implement throttling

The following policy limits access to **10 requests per minute**. Paste the XML in: API Management Service > `nicold-apim` > APIs > All APIs > `/` > all operations > inbound processing > policies (code editor)

```xml
<policies>
    <inbound>
        <base />
        <rate-limit calls="10" renewal-period="60" />
        <set-backend-service id="apim-generated-policy" backend-id="apim-001" />
    </inbound>
    <backend>
        <base />
    </backend>
    <outbound>
        <base />
    </outbound>
    <on-error>
        <base />
    </on-error>
</policies>

```

To limit at 2 calls **per IP** in 60 seconds, use the following rate-limit xml:
```xml
<rate-limit-by-key calls="2" renewal-period="60" counter-key="@(context.Request.IpAddress)" />
```

To limit at 2 calls per API KEY in 60 seconds, use the following rate-limit xml:
```xml
<rate-limit-by-key calls="2" renewal-period="60" counter-key="@(context.Request.Headers.GetValueOrDefault("api-key"))" />
```

# Show in a Response header (_aoai-origin_) the host of the OpenAI API Called

Add the following XML in the **outbound** policy:

```xml
<outbound>
    <base />
    <set-header name="aoai-origin" exists-action="override">
        <value>@(context.Request.Url.Host)</value>
    </set-header>
</outbound>
```
# Round robin calls between 2 instances of Open AI

Use the following **inbound** policy:

```xml
<inbound>
    <base />
    <cache-lookup-value key="backend-rr" variable-name="backend-rr" />
    <choose>
        <when condition="@(!context.Variables.ContainsKey("backend-rr"))">
            <set-variable name="backend-rr" value="0" />
            <cache-store-value key="backend-rr" value="0" duration="100" />
        </when>
    </choose>
    <choose>
        <when condition="@(Convert.ToInt32(context.Variables["backend-rr"]) == 0)">
            <set-backend-service backend-id="apim-001" />
            <set-variable name="backend-rr" value="1" />
            <cache-store-value key="backend-rr" value="1" duration="100" />
        </when>
        <otherwise>
            <set-backend-service backend-id="apim-002" />
            <set-variable name="backend-rr" value="0" />
            <cache-store-value key="backend-rr" value="0" duration="100" />
        </otherwise>
    </choose>
</inbound>
```
> 💥In a round robin scenario, inorder to work properli both open AI instances must have same deployments

# Fallback on a second openAI instance for 30 secs. If the first send a 429 error (too many requests)

TODEBUG!!!


<policies>
    <inbound>
        <base />
        <cache-lookup-value key="useSecondaryBackend" variable-name="useSecondaryBackend" />
        <choose>
            <when condition="@(context.Variables["useSecondaryBackend"] != null)">
                <set-backend-service backend-id="apim-002" />
            </when>
            <otherwise>
                <set-backend-service backend-id="apim-001" />
            </otherwise>
        </choose>
    </inbound>
    <backend>
        <base />
    </backend>
    <outbound>
        <base />
    </outbound>
    <on-error>
        <base />
        <set-header name="aoai-origin" exists-action="override">
            <value>@(context.Request.Url.Host)</value>
        </set-header>
        <choose>
            <when condition="@(context.Response.StatusCode >= 400 && context.Response.StatusCode < 500)">
				<set-variable name="useSecondaryBackend" value="true" />
                <cache-store-value key="useSecondaryBackend" value="true" duration="60" />
            </when>
        </choose>
    </on-error>
</policies>

# Generate a report of API Usage by access key
Go to: API Management Services > `nicold-apim` > logs > New QUery (KQL mode):

```
ApiManagementGatewayLogs
| where ApimSubscriptionId != ''
| summarize count() by ApimSubscriptionId
| order by count_ desc
| render table
```

Pie chart

```
ApiManagementGatewayLogs
| where ApimSubscriptionId != ''
| summarize  count() by ApimSubscriptionId 
| order by count_ desc 
| render piechart 
```

![pie chart](../../assets/post/2025/apim-aoai/02-pie-chart.png)


# Generate a report of API Usage by source IP
Go to: API Management Services > `nicold-apim` > logs > New QUery (KQL mode):

```
ApiManagementGatewayLogs
| where ApimSubscriptionId != ''
| summarize count() by CallerIpAddress
| order by count_ desc
| render table

``` 

# Generate a report of backend usage over time

Go to: API Management Services > `nicold-apim` > logs > New QUery (KQL mode):

Group by hour

```
ApiManagementGatewayLogs 
| where ApimSubscriptionId != '' 
| summarize  count() by bin (TimeGenerated, 1h), BackendId 
| render timechart

```
Grouped by day
 
```
ApiManagementGatewayLogs 
| where ApimSubscriptionId != '' 
| summarize  count() by bin (TimeGenerated, 1d), BackendId 
| render timechart
```

![time chart](../../assets/post/2025/apim-aoai/03-usage-over-time.png)

Column chart

```
ApiManagementGatewayLogs
| where ApimSubscriptionId != ''
| summarize count() by bin(TimeGenerated, 1h), BackendId
| project TimeGenerated, BackendId, count_
| render columnchart  kind=stacked
```

# Generate a report of API requests by OpenAI deployment-id used

Go to: API Management Services > `nicold-apim` > logs > New QUery (KQL mode):

```
ApiManagementGatewayLogs
| extend DeploymentSubstring = extract(@"deployments/([^/]+)", 1, Url)
| where ApimSubscriptionId != ''
| summarize count() by DeploymentSubstring
| render table
```

[def]: #recipes-list