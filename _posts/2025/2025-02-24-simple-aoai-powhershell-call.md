---
title: Azure OpenAI powershell chat-completition call sample
date: 2025-02-24 10:00
tags: [Azure, powershell, OpenAI ]
excerpt: "sample to copy and paste"

header:
  overlay_image: https://live.staticflickr.com/65535/52755090506_6cf0808a3c_h.jpg
  caption: "Photo credit: [**nicola since 1972**](https://www.flickr.com/photos/15216811@N06/52755090506)"
---

The following powershell samples shows how to call an Azure OpenAI chat completition endpoint API

```
# Azure OpenAI metadata variables
$openai = @{
   api_key     = "YOUR_APIKEY_HERE"
   api_base    = "https://your-enpoint-here.openai.azure.com/" # your endpoint
   api_version = '2024-02-01'
   name        = 'your-deployment-name-here' # custom name you chose for your deployment
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
$url = "$($openai.api_base)/openai/deployments/$($openai.name)/chat/completions?api-version=$($openai.api_version)"
$response = IRM -Uri $url -Headers $headers -Body $body -Method Post -ContentType 'application/json'
$response.choices.message.content

```