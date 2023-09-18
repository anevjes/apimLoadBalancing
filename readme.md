```xml
<!--
    IMPORTANT:
    - Policy elements can appear only within the <inbound>, <outbound>, <backend> section elements.
    - To apply a policy to the incoming request (before it is forwarded to the backend service), place a corresponding policy element within the <inbound> section element.
    - To apply a policy to the outgoing response (before it is sent back to the caller), place a corresponding policy element within the <outbound> section element.
    - To add a policy, place the cursor at the desired insertion point and select a policy from the sidebar.
    - To remove a policy, delete the corresponding policy statement from the policy document.
    - Position the <base> element within a section element to inherit all policies from the corresponding section element in the enclosing scope.
    - Remove the <base> element to prevent inheriting policies from the corresponding section element in the enclosing scope.
    - Policies are applied in the order of their appearance, from the top down.
    - Comments within policy elements are not supported and may disappear. Place your comments between policy elements or at a higher level scope.
-->
<policies>
    <inbound>
        <base />
        <cache-lookup-value key="backend-counter" variable-name="backend-counter" />
        <choose>
            <when condition="@(!context.Variables.ContainsKey("backend-counter"))">
                <set-variable name="backend-counter" value="0" />
                <cache-store-value key="backend-counter" value="0" duration="100" />
            </when>
        </choose>
        <choose>
            <when condition="@(int.Parse((string)context.Variables["backend-counter"]) == 0)">
                <set-backend-service backend-id="openAiBackend1" />
                <set-variable name="backend-counter" value="1" />
                <cache-store-value key="backend-counter" value="1" duration="100" />
            </when>
            <when condition="@(int.Parse((string)context.Variables["backend-counter"]) == 1)">
                <set-backend-service backend-id="openAiBackend2" />
                <set-variable name="backend-counter" value="2" />
                <cache-store-value key="backend-counter" value="2" duration="100" />
            </when>
            <otherwise>
                <set-backend-service backend-id="openAiBackend3" />
                <set-variable name="backend-counter" value="0" />
                <cache-store-value key="backend-counter" value="0" duration="100" />
            </otherwise>
        </choose>
    </inbound>
    <backend>
        <retry condition="@(context.Response.StatusCode == 429)" count="3" interval="10" first-fast-retry="true">
            <choose>
                <when condition="@(context.Response.StatusCode == 429)">
                    <choose>
                        <when condition="@(int.Parse((string)context.Variables["backend-counter"]) == 0)">
                            <set-backend-service backend-id="openAiBackend1" />
                            <set-variable name="backend-counter" value="1" />
                            <cache-store-value key="backend-counter" value="1" duration="100" />
                        </when>
                        <when condition="@(int.Parse((string)context.Variables["backend-counter"]) == 1)">
                            <set-backend-service backend-id="openAiBackend2" />
                            <set-variable name="backend-counter" value="2" />
                            <cache-store-value key="backend-counter" value="2" duration="100" />
                        </when>
                        <otherwise>
                            <set-backend-service backend-id="openAiBackend3" />
                            <set-variable name="backend-counter" value="0" />
                            <cache-store-value key="backend-counter" value="0" duration="100" />
                        </otherwise>
                    </choose>
                </when>
            </choose>
            <forward-request buffer-request-body="true" />
        </retry>
    </backend>
    <outbound>
        <base />
        <set-header name="Next-Backend-Counter" exists-action="override">
            <value>@((string)context.Variables["backend-counter"])</value>
        </set-header>
        <!-- Uncomment for debugging, this outputs the next backend-counter in the JSON response body. -->
        <!--<set-body>@{ 
            JObject body = context.Response.Body.As<JObject>(); 
            body.Add(new JProperty("backend-counter", ((string)context.Variables["backend-counter"])));
            return body.ToString(); 
        }</set-body>-->
    </outbound>
    <on-error>
        <base />
    </on-error>
</policies>
```
