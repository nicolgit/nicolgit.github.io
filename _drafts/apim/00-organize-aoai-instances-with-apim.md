---
title: Distribute Azure Open AI deployment in enterprise using Azure API Management service
date: 2025-01-01 10:00
tags: [Azure, networking, API manager, OpenAI ]
excerpt: ""

header:
  overlay_image: https://live.staticflickr.com/65535/52755090506_6cf0808a3c_h.jpg
  caption: "Photo credit: [**nicola since 1972**](https://www.flickr.com/photos/15216811@N06/52755090506)"
---

Azure OpenAI Service provides REST API access to OpenAI's powerful language models including o1, o1-mini, GPT-4o, GPT-4o mini, GPT-4 Turbo with Vision, GPT-4, GPT-3.5-Turbo, and Embeddings model series. These models can be easily adapted to your specific task including but not limited to content generation, summarization, image understanding, semantic search, and natural language to code translation.

Azure API Management, on the other hand, is a comprehensive API management platform that helps enterprises manage, secure, and monitor their APIs. It provides features such as API gateway, rate limiting, analytics, and developer portal, making it easier for businesses to expose their services to internal and external consumers.

By using Azure API Manager in front of Azure Open AI instances, enterprises can ensure better control, security, and visibility over their AI deployments. This setup allows for centralized management of API traffic, improved performance through caching and load balancing, and enhanced security with features like authentication and authorization.

Combining Azure Open AI with Azure API Manager enables enterprises to efficiently distribute and manage their AI capabilities while maintaining high standards of security and performance.

----------------------------

obiettivo di questa serie di post é quello di mostrate alcuni dei pattern da usare per esporre Azure Open AI attraverso APIM. 

In this context, the following architecturea and implementation in Azure Architecture Center are **developed** and **maintained** by Azure Pattern and Practices team. I suggest yout to bookmark them for further information.

* https://learn.microsoft.com/en-us/azure/architecture/ai-ml/architecture/azure-openai-baseline-landing-zone
* https://github.com/Azure-Samples/azure-openai-chat-baseline-landing-zone

in questi post ci focalizzeremo sulla configurazione delle policy e di implementazione di alcuni pattern di utilizzo con esse. La parte di configurazione della rete e l'integrazione con una ESLZ é out of scope. Per un walktrough che guidi l'integrazione di APIM ed AOAI in un contesto di hub & spoke, è possibile fare riferimento anche [al mio articolo](https://github.com/nicolgit/hub-and-spoke-playground/blob/main/scenarios/aoai.md) disponibile nell'ambito del progetto [the hub-and-spoke playground](https://github.com/nicolgit/hub-and-spoke-playground)


Il laboratorio su cui lavoró prevede le seguenti risorse:

* Azure API Management service (developer SKU) enterpriseapim
* 2 x Azure OpenAI service (S0 SKU) apimaoai01 and apimaoai02

come mostrato nello schema seguente:

«««««««««««««««««««««««««««««««« ARCHITETTURA SUPER SEMPLICE »»»»»»»»»»»»»»»»»»»»»»»»»»»»»

# 01 Add Azure open AI as backend resource

Go to API Management services > `nicold-apim` > Backends > Add
* Name: `apim-001`
* Backend hosting type: Custom URL
* Runtime URL: https://nicold-aoai-001.openai.azure.com 
* Authorization credential
  * Headers
    * Name: `api-key`
    * Key: _your endpoint key_
* click [create]

Go to API Management services > `nicold-apim` > Backends > Add
* Name: `apim-002`
* Backend hosting type: Custom URL
* Runtime URL: https://nicold-aoai-002.openai.azure.com 
* Authorization credential
  * Headers
    * Name: `api-key`
    * Key: _your endpoint key_
* click [create]

here the result:
![backend resorces added](../../assets/post/2025/apim-aoai/01-add-backend-resources.png)

# 02 


- esporre una API attraverso policy
  - livello root
  - in /aoai (2 instance) (instance/deployment...)
  




(forse questi non servono più)

# More information
* Azure Open AI Services: <https://learn.microsoft.com/en-us/azure/ai-services/openai/overview>
* Azure API Management: <https://learn.microsoft.com/en-us/azure/api-management/api-management-key-concepts>



* Azure APIM to scale AOAI - https://github.com/Azure/aoai-apim 
* Manage Azure OpenAI using APIM - https://github.com/microsoft/AzureOpenAI-with-APIM
