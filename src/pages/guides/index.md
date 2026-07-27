---
title: Guides - AEM
description: This is the guides overview page of AEM
---

## Introduction

Adobe Experience Manager (AEM) is a powerful platform for organizations to customize solutions that span content management, digital assets management, and forms management. [Learn more](https://business.adobe.com/products/experience-manager/adobe-experience-manager.html) about AEM.

Browse the [AEM a Cloud Service documentation](https://experienceleague.adobe.com/docs/experience-manager-cloud-service/content/home.html) or the [AEM 6.5 documentation](https://experienceleague.adobe.com/docs/experience-manager-65.html).

## Articles

### Configuring the OpenAPI-based APIs

Before invoking the OpenAPI-based APIs, create an Adobe Developer Console project mapping one or more APIs to specific environments it can access. Then register the Adobe Developer Console project's client ID with the intended AEM Cloud Service environment.

[Learn more](https://experienceleague.adobe.com/en/docs/experience-manager-cloud-service/content/implementing/developing/open-api-based-apis#configuring-api-access)

### Using the OpenAPI-based APIs

Program with AEM as a Cloud Service's OpenAPI-based APIs, following patterns including authentication, error handling, and selecting between stable and experimental versions.

[Learn more](how-to/index.md)

### AEM Events

AEM as a Cloud Service offers a cloud-native solution for AEM expandability. Develop your extension and let it get triggered by events.

[Learn more](events/index.md)

### Tutorial: Server-to-Server authentication with OpenAPI-based APIs

[Follow a tutorial](https://experienceleague.adobe.com/en/docs/experience-manager-learn/cloud-service/aem-apis/openapis/invoke-api-using-oauth-s2s) about how to configure and invoke OpenAPI-based APIs using OAuth Server-to-Server authentication, intended for backend services needing API access without user interaction. You will develop a sample NodeJS application that calls the Assets Author API to retrieve asset metadata.

### Tutorial: Web App authentication with OpenAPI-based APIs

[Follow a tutorial](https://experienceleague.adobe.com/en/docs/experience-manager-learn/cloud-service/aem-apis/invoke-openapi-based-aem-apis-from-web-app) about how to configure and invoke OpenAPI-based APIs using an external web app that performs OAuth Web App authentication, intended for web applications with frontend and backend components needing to access AEM APIs on behalf of a user. You will develop a sample NodeJS application representing a PIM (Product Information Management) solution that interacts with an AEM Author environment.

### Tutorial: Single-Page App authentication with OpenAPI-based APIs

[Follow a tutorial](https://experienceleague.adobe.com/en/docs/experience-manager-learn/cloud-service/aem-apis/openapis/invoke-api-using-oauth-single-page-app) about configuring and invoking OpenAPI-based APIs from a purely front-end application like an SPA that performs OAuth Single-Page auth. This method is intended for JavaScript-based applications running in the browser or similar scenarios where there is no backend server or you need to invoke AEM APIs from the client side on behalf of a user.
You will develop a sample ReactJS application representing an SPA that interacts with an AEM Author environment to fetch Content Fragment Model and DAM folders.

### Tutorial: Using OpenAPI-based APIs and Events

Take a look at [AEM Assets events for PIM integration](https://experienceleague.adobe.com/en/docs/experience-manager-learn/cloud-service/aem-eventing/examples/assets-pim-integration) to learn the steps that are needed to onboard the OpenAPI-based APIs and IO Events for the use case of importing metadata from a third-party system whenever a new asset is uploaded into the DAM. This is a great walkthrough for this use case, but also as an example of how to get started with these new development patterns.

### Metadata Schemas developer guide

Metadata schemas specify the metadata properties that matter to your organization, establishing their names, types, constraints, and other details. This guide supplements the Metadata Schema API documentation, covering how to structure a schema, how to use the available keywords, and how a schema shapes what you read and write through the Metadata API for assets and other entities.

[Learn more](metadata-schema/index.md)
