# Document Intelligence + API Management DevOps

This repo houses a quick demonstration for mediating Azure Document Intelligence custom model ids via an API Gateway (Azure API Management).

## Configuration Steps

1. [Import](https://learn.microsoft.com/en-us/azure/api-management/import-api-from-oas?tabs=portal) OpenAPI specification from the [Azure REST API / SDK library](https://azure.github.io/azure-sdk/releases/latest/all/specs.html) into API Management.  Be sure to choose the appropriate API version for your needs.
2. Identify which API operations you need to mediate for your consumers.  In most cases, consumers will call Analyze Document and Get Analyze Results.  In both cases, the Model ID is part of the URL template.
3. For each operation, we'll need to remove / replace the model ID template parameter.  To do this, click through to the API Operation and click the pencil icon under Frontend.  This will allow us to [edit the API configuration](https://learn.microsoft.com/en-us/azure/api-management/edit-api).
4. In the Frontend configuration screen, modify the URL to remove the parameter "{modelId}" and replace it with the word "model".  Delete the parameter "modelId" from the Template parameters tab below.
5. Before moving on, we're going to create 2 [named values](https://learn.microsoft.com/en-us/azure/api-management/api-management-howto-properties?tabs=azure-portal) to hold some configuration.  Navigate to the Named Values section of the APIM control plane.
6. Create 2 named values, one to hold your API key and one to hold the model ID.  Name them however you like, but remember the name for future use.
7. Go back to the API operation and now click on the HTML icon ```</>``` under Inbound processing.  We need to add [policies](https://learn.microsoft.com/en-us/azure/api-management/api-management-howto-policies) to add in a model ID to replace the parameter we removed.
8. We will add 2 policies:
    1. A [```rewrite-uri```](https://learn.microsoft.com/en-us/azure/api-management/rewrite-uri-policy) policy to substitute our model id value
    2. A [```set-header```](https://learn.microsoft.com/en-us/azure/api-management/set-header-policy) policy to make sure our request to the backend has the correct API key included
9.  Your policy document should look like this (replace the values in curly braces {{ }} with the appropriate names you gave in step 6):

```xml

<!--
    - Policies are applied in the order they appear.
    - Position <base/> inside a section to inherit policies from the outer scope.
    - Comments within policies are not preserved.
-->
<!-- Add policies as children to the <inbound>, <outbound>, <backend>, and <on-error> elements -->
<policies>
    <!-- Throttle, authorize, validate, cache, or transform the requests -->
    <inbound>
        <base />
        <rewrite-uri template="documentintelligence/documentModels/{{doc-intel-model-id}}:analyze" copy-unmatched-params="true" />
        <set-header name="Ocp-Apim-Subscription-Key" exists-action="override">
            <value>{{doc-intel-key}}</value>
        </set-header>
    </inbound>
    <!-- Control if and how the requests are forwarded to services  -->
    <backend>
        <base />
    </backend>
    <!-- Customize the responses -->
    <outbound>
        <base />
    </outbound>
    <!-- Handle exceptions and customize error responses  -->
    <on-error>
        <base />
    </on-error>
</policies>

```
10. Once this is complete, the APIM instance is ready to receive configured model IDs from CI/CD.
11. To update the model ID configured in APIM, all we need to do is update the named value we created.  To do this, we call an [Azure CLI command](https://learn.microsoft.com/en-us/cli/azure/apim/nv?view=azure-cli-latest): ```az apim nv update --service-name yourapimname -g yourresourcegroup --named-value-id yournamedvalue -value yournewmodelid```.  This can be inserted into any pipeline where it makes sense to make this change.  It could be during model creation or after as a secondary step.
12. An [example GitHub Action](https://github.com/mitulashah/demo-doc-intel-devops/blob/main/.github/workflows/update.yml) with the required configuration can be found in this repository.

## Caveats
- There is a possibility that mediating the Document Intellgence API in this way could break SDK compatibility.  Since the Model ID is included as a URL template, SDKs likely will require that parameter to be defined.  In some cases passing the replacement value we added "model" might work.  It would entirely depend on how the SDK builds the URL to Doc Intelligence and may vary depending on which SDK is used.
- This solution does not address scaling/load balancing across multiple instances.  Generally, the Document Intelligence servince can be scaled to meet your needs.  [Contact support](https://learn.microsoft.com/en-us/azure/ai-services/document-intelligence/service-limits?view=doc-intel-4.0.0#increasing-transactions-per-second-request-limit) to request additional capacity.
