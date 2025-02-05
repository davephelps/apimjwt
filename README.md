# How to Read Claims from a JWT in Azure API Management

Sample that shows how to use Azure API Management Policy to read JWT values and optionally write to Azure Application Insights.

The following example does a few things:
* Validates the JWT token passed to APIM, in this case from Microsoft Entra ID, including the *audience* and and *role* claims. For this to work, the client must have already been granted permission and of course requested an access token
* Read the JWT token using Azure APIM expressions, then extracts the *appid* claim into a variable
* Writes the *appid* value to the Application Insights *Traces* table using the *Trace* policy so it can be reported on
* The *context.User* expressions are only populated if the subscription key is passed in the request. This isn't mandatory, so if only using JWT for example, extracting details from the JWT can be useful for rate limiting or logging purposes

```xml
<policies>
    <inbound>
        <base />
        <validate-jwt header-name="Authorization" failed-validation-httpcode="401" failed-validation-error-message="Unauthorized" require-expiration-time="true" require-scheme="Bearer" require-signed-tokens="true">
            <openid-config url="https://login.microsoftonline.com/your_tenant/.well-known/openid-configuration" />
            <required-claims>
                <claim name="aud" match="all">
                    <value>https://yourtenant/api/retail</value>
                </claim>
                <claim name="roles" match="all">
                    <value>contosoretail.write</value>
                </claim>
            </required-claims>
        </validate-jwt>
        <!-- App Id -->
        <set-variable name="jwt_appid" value="@(context.Request.Headers.GetValueOrDefault("Authorization","").AsJwt().Claims.GetValueOrDefault("appid"))" />
        <!-- Log user details -->
        <trace source="userDetails" severity="information">
            <message>@{
                return new JObject(
                    new JProperty("appid", context.Variables["jwt_appid"]),
                    new JProperty("email",context.User.Email),
                    new JProperty("firstName",context.User.FirstName),
                    new JProperty("lastName",context.User.LastName)
                ).ToString();
            }</message>
        </trace>
        <set-header name="Authorization" exists-action="delete" />
        <!-- Managed Identity -->
        <authentication-managed-identity client-id="user managed identity" resource="https://yourtenant.onmicrosoft.com/contoso/backend" />
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
To read the *appid* from Application Insights, find the relevant entry from the *requests* table and join on *traces*:
``` sql
requests
| where name =='POST /contoso/order/oauth'
| join kind=leftouter (
    traces
|   extend body=todynamic(tostring(message))
|   extend appid= body.appid
  ) on operation_Id
| project timestamp, name, operation_Id, appId
| order by timestamp desc  
```
