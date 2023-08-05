# Overview
It appears that dotnet publish directly to a docker registry using Microsoft.NET.Build.Containers __fails with an unauthorized error if you log into docker in certain situations (such as when an identity token or credstore is used with an Azure Container Registry)__

In a GHA workflow, I had this process:
- az login via federated credentials
   ```yaml
    - uses: azure/login@v1
      name: Sign in to Azure
      with:
        client-id: ${{ secrets.AZURE_CLIENT_ID }}
        tenant-id: ${{ secrets.AZURE_TENANT_ID }}
        subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
   ```
- log into Azure Container Registry
   ```yaml
    - name: Login to ACR
      run: |
        az acr login --name <my-registry>
   ```
- publish image to ACR using Microsoft.NET.Build.Containers
   ```yaml
    - name: publish
      run: dotnet publish ./my/path/to.csproj --os linux --arch x64 /t:PublishContainer -c Release
   ```
- this then resulted in a 401 unauthorized error on the publish step:
  ```txt
  dotnet publish ./my/path/to.csproj --os linux --arch x64 /t:PublishContainer -c Release
  shell: /usr/bin/bash -e {0}
  env:
    DOTNET_ROOT: /usr/share/dotnet
    AZURE_HTTP_USER_AGENT: 
    AZUREPS_HOST_ENVIRONMENT: 
  ##[endgroup]
  MSBuild version 17.6.8+c70978d4d for .NET
    Determining projects to restore...
    Restored /home/runner/work/some/path/my/path/to.csproj (in 224 ms).
    to -> /home/runner/work/some/path/to/bin/Release/net7.0/linux-x64/to.dll
    to -> /home/runner/work/some/path/to/bin/Release/net7.0/linux-x64/publish/
    Building image 'my-image-name' with tags 1.0.0 on top of base image mcr.microsoft.com:443/dotnet/runtime:7.0
    Uploading layer sha256:1d5252f66ea9b661aceca1027b3d7ca259a50608261a25b51148119ecf086932 to my-registry.azurecr.io
    Uploading layer sha256:75b8c875f256d7ad2c486a3a53c3bdf6b084bfd885e2ea845d54aab237e16b76 to my-registry.azurecr.io
    Uploading layer sha256:3b04a5dc83ef9b6e55e871d3f28543d55da237e04fe428844764759854f62b4a to my-registry.azurecr.io
    Uploading layer sha256:6703541983d9dc7ce7f7b2db5cc628f5c0b0f4a6e38db93d1a344f40c6f6f153 to my-registry.azurecr.io
    Uploading layer sha256:53d0938d83e0c6d4497a4e4fa72a65a7767912f3cff06399ddf63b7ee302c618 to my-registry.azurecr.io
  ##[error]/home/runner/.nuget/packages/microsoft.net.build.containers/7.0.306/build/Microsoft.NET.Build.Containers.targets(195,5): error CONTAINER1013: Failed to push to the output registry: System.Net.Http.HttpRequestException: Response status code does not indicate success: 401 (Unauthorized). [/home/runner/work/some/path/my/path/to.csproj]
  ##[error]/home/runner/.nuget/packages/microsoft.net.build.containers/7.0.306/build/Microsoft.NET.Build.Containers.targets(195,5): error CONTAINER1013:    at System.Net.Http.HttpResponseMessage.EnsureSuccessStatusCode() [/home/runner/work/some/path/my/path/to.csproj]
  ##[error]/home/runner/.nuget/packages/microsoft.net.build.containers/7.0.306/build/Microsoft.NET.Build.Containers.targets(195,5): error CONTAINER1013:    at Microsoft.NET.Build.Containers.AuthHandshakeMessageHandler.GetAuthenticationAsync(String registry, String scheme, Uri realm, String service, String scope, CancellationToken cancellationToken) in /_/src/Containers/Microsoft.NET.Build.Containers/AuthHandshakeMessageHandler.cs:line 138 [/home/runner/work/some/path/my/path/to.csproj]
  ##[error]/home/runner/.nuget/packages/microsoft.net.build.containers/7.0.306/build/Microsoft.NET.Build.Containers.targets(195,5): error CONTAINER1013:    at Microsoft.NET.Build.Containers.AuthHandshakeMessageHandler.SendAsync(HttpRequestMessage request, CancellationToken cancellationToken) in /_/src/Containers/Microsoft.NET.Build.Containers/AuthHandshakeMessageHandler.cs:line 182 [/home/runner/work/some/path/my/path/to.csproj]
  ##[error]/home/runner/.nuget/packages/microsoft.net.build.containers/7.0.306/build/Microsoft.NET.Build.Containers.targets(195,5): error CONTAINER1013:    at System.Net.Http.HttpClient.<SendAsync>g__Core|83_0(HttpRequestMessage request, HttpCompletionOption completionOption, CancellationTokenSource cts, Boolean disposeCts, CancellationTokenSource pendingRequestsCts, CancellationToken originalCancellationToken) [/home/runner/work/some/path/my/path/to.csproj]
  ##[error]/home/runner/.nuget/packages/microsoft.net.build.containers/7.0.306/build/Microsoft.NET.Build.Containers.targets(195,5): error CONTAINER1013:    at Microsoft.NET.Build.Containers.Registry.BlobAlreadyUploadedAsync(String repository, String digest, HttpClient client, CancellationToken cancellationToken) in /_/src/Containers/Microsoft.NET.Build.Containers/Registry.cs:line 497 [/home/runner/work/some/path/my/path/to.csproj]
  ##[error]/home/runner/.nuget/packages/microsoft.net.build.containers/7.0.306/build/Microsoft.NET.Build.Containers.targets(195,5): error CONTAINER1013:    at Microsoft.NET.Build.Containers.Registry.<>c__DisplayClass51_0.<<PushAsync>b__0>d.MoveNext() in /_/src/Containers/Microsoft.NET.Build.Containers/Registry.cs:line 546 [/home/runner/work/some/path/my/path/to.csproj]
  ##[error]/home/runner/.nuget/packages/microsoft.net.build.containers/7.0.306/build/Microsoft.NET.Build.Containers.targets(195,5): error CONTAINER1013: --- End of stack trace from previous location --- [/home/runner/work/some/path/my/path/to.csproj]
  ##[error]/home/runner/.nuget/packages/microsoft.net.build.containers/7.0.306/build/Microsoft.NET.Build.Containers.targets(195,5): error CONTAINER1013:    at Microsoft.NET.Build.Containers.Registry.PushAsync(BuiltImage builtImage, ImageReference source, ImageReference destination, Action`1 logProgressMessage, CancellationToken cancellationToken) in /_/src/Containers/Microsoft.NET.Build.Containers/Registry.cs:line 575 [/home/runner/work/some/path/my/path/to.csproj]
  ##[error]/home/runner/.nuget/packages/microsoft.net.build.containers/7.0.306/build/Microsoft.NET.Build.Containers.targets(195,5): error CONTAINER1013:    at Microsoft.NET.Build.Containers.Tasks.CreateNewImage.ExecuteAsync(CancellationToken cancellationToken) in /_/src/Containers/Microsoft.NET.Build.Containers/Tasks/CreateNewImage.cs:line 127 [/home/runner/work/some/path/my/path/to.csproj]
  ##[error]Process completed with exit code 1.
  ```

I was then also able to reproduce the same error on a local Alpine Linux docker image logging in via clientId and clientSecret instead of federated credentials:
- login to azure `az login --service-principal -u <clientId> -p <clientSecret> -t <azure ad tenant guid>`
- login to acr `az acr login --name <my-registry>`
- attempt to publish to acr `dotnet publish ./my/path/to.csproj --os linux --arch x64 /t:PublishContainer -c Release`
- received the exact same error

# Reproduction steps
Ran on Windows OS in Windows Powershell (terminal tool)
### Prerequisite - creating an Azure Container Registry and service principal with permissions for pushing
1. Have an Azure account
2. Have Az CLI installed
3. Have Docker CLI installed
4. `az login`
5. Make variables 
   ```shell
   $chars = 'abcdefghijklmnopqrstuvwxyz'
   $RANDOM_SUFFIX = ''
   for ($i = 0; $i -lt 15; $i++) {
       $random = Get-Random -Minimum 0 -Maximum $chars.Length
       $RANDOM_SUFFIX += $chars[$random]
   }
   $RESOURCE_GROUP_NAME = "acr-docker-dotnetcontainers-bug"
   $ACR_NAME = "testacr$RANDOM_SUFFIX"
   $SERVICE_PRINCIPAL_NAME = "acr-docker-dotnetcontainers-bug-user"
   $LOCATION = "centralus"
   $SUBSCRIPTION_ID = (az account show --query id --output tsv)
   ```
6. Create a poc resource group - `az group create --name $RESOURCE_GROUP_NAME --location $LOCATION`
7. Create an ACR
   ```shell
   az acr create --name $ACR_NAME --resource-group $RESOURCE_GROUP_NAME --sku Basic --location $LOCATION
   $ACR_ID=$(az acr show --name $ACR_NAME --query id --output tsv)
   ```
7. Create a service principal user that can log into azure for this resource group -
   ```shell
   $servicePrincipal = az ad sp create-for-rbac --name $SERVICE_PRINCIPAL_NAME --role Contributor --scopes /subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP_NAME
   $appId = $servicePrincipal | ConvertFrom-Json | Select-Object -ExpandProperty appId
   $password = $servicePrincipal | ConvertFrom-Json | Select-Object -ExpandProperty password
   $tenant = $servicePrincipal | ConvertFrom-Json | Select-Object -ExpandProperty tenant
   ```
8. Grant push access to ACR - `az role assignment create --assignee $appId --scope $ACR_ID --role AcrPush`
9. Output vars
   ```shell
   Write-Host "ACR Registry Name: $ACR_NAME.azurecr.io"
   Write-Host "Service Principal ClientId: $appId"
   Write-Host "Service Principal ClientSecret (Password): $password"
   Write-Host "Service Principal Tenant (Password): $tenant"
   ```
10. Test credentials (login to azure, acr, and docker push a test image)
   ```shell
   az logout
   az login --service-principal -u $appId -p $password -t $tenant
   az acr login --name $ACR_NAME
   docker pull hello-world
   docker tag hello-world:latest "$ACR_NAME.azurecr.io/hello-world:latest"
   docker push "$ACR_NAME.azurecr.io/hello-world:latest"
   $images = az acr repository list --name $ACR_NAME --output tsv
   if ($images -contains "hello-world") {
       Write-Host "Image push to ACR was successful!"
   } else {
       Write-Host "Image push to ACR failed!"
   }
   ```
11. We now have a service principal login and verified it's able to docker push to our ACR
12. Cleanup (after observing the bug below) (az logout and az login as your original user)
   ```shell
   az group delete --name $RESOURCE_GROUP_NAME --yes --no-wait
   az ad sp delete --id $appId
   ```

### Reproduce the bug by attempting to push to ACR through Microsoft.NET.Build.Containers
We'll follow https://learn.microsoft.com/en-us/dotnet/core/docker/publish-as-container

1. Az CLI installed
2. Have Docker installed and Daemon running
3. Have .NET 7+ SDK installed
4. Navigate to a directory suitable for a new .NET project
5. Create the .NET project 
   ```shell
   dotnet new worker -o Worker -n DotNet.ContainerImage
   cd .\Worker\
   dotnet add package Microsoft.NET.Build.Containers
   ```
6. Test and verify success of the Microsoft.NET.Build.Containers publish to your local docker daemon - `dotnet publish --os linux --arch x64 /t:PublishContainer -c Release`
7. Ensure you're logged into ACR (see **Prerequisite - creating an Azure Container Registry and service principal with permissions for pushing**)
8. Open `DotNet.ContainerImage.csproj` and add your registry to the csproj to push to ACR instead of local docker daemon (run `Write-Host "ACR Registry Name: $ACR_NAME.azurecr.io"` from above):
   ```xml
   <PropertyGroup>
      <ContainerRegistry><registry>.azurecr.io</ContainerRegistry>
   </PropertyGroup>
   ```
9. Repush and observe the bug - `dotnet publish --os linux --arch x64 /t:PublishContainer -c Release` - `error CONTAINER1013: Failed to push to the output registry: System.Net.Http.HttpRequestException: Response status code does not indicate success: 401 (Unauthorized)`
10. (repeat from prerequisites) Double check that we CAN push to ACR bypassing Microsoft.NET.Build.Containers by using docker directly - `docker push "$ACR_NAME.azurecr.io/hello-world:latest"`

# Work arounds
### After `az acr login` manually change the `$HOME/.docker/config.json` and merge identitytoken and auth entries
This is the approach I did and am keeping until a fix is in place. Maybe easier workarounds but I was still figuring out the issue. Perhaps Microsoft.NET.Build.Containers does super custom docker commaneds and doesn't support an identity token, only basic auth?

- `az login` using **service principal or federated credentials**
- `az acr login --name your-registry`, which creates the following docker auth config at `$HOME/.docker/config.json`. The `auth` is base64'd `00000000-0000-0000-0000-000000000000:`:
   ```json
   {
        "auths": {
                "your-registry.azurecr.io": {
                        "auth": "MDAwMDAwMDAtMDAwMC0wMDAwLTAwMDAtMDAwMDAwMDAwMDAwOg==",
                        "identitytoken": "REDACTED JWT"
                }
   }
   ```
- We then manually update `$HOME/.docker/config.json` and overrwrite `auths.[your-registry.azurecr.io].auth` and set it to the base64 of `00000000-0000-0000-0000-000000000000:REDACTED JWT`, and also remove the `identitytoken` element:
   ```json
   {
        "auths": {
                "your-registry.azurecr.io": {
                        "auth": "<base 64 of '00000000-0000-0000-0000-000000000000:REDACTED JWT'>"
                }
   }
   ```
- We then proceed with dotnet publish using Microsoft.NET.Build.Containers and successfully push to ACR

Here is the updated Github Actions workflow file to automate this process:
- az login via federated credentials
   ```yaml
    - uses: azure/login@v1
      name: Sign in to Azure
      with:
        client-id: ${{ secrets.AZURE_CLIENT_ID }}
        tenant-id: ${{ secrets.AZURE_TENANT_ID }}
        subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
   ```
- log into Azure Container Registry
   ```yaml
    - name: Login to ACR
      run: |
        az acr login --name <my-registry>
   ```
- (NEW!) adjust the docker auth file so we can successfuly push
   ```yaml
    - name: Update docker auth to circumvent docker/acr/Microsoft.NET.Build.Containers service principal login bug
      env:
        REGISTRY_URL: <my-registry>
      run: |
        JWT=$(jq -r '.auths."'$REGISTRY_URL'".identitytoken' "$HOME/.docker/config.json")
        AUTH_STRING=$(echo -n "00000000-0000-0000-0000-000000000000:$JWT" | base64 | tr -d '\n')

        jq '.auths."'$REGISTRY_URL'" |= . + {"auth": "'$AUTH_STRING'"} | del(.auths."'$REGISTRY_URL'".identitytoken)' "$HOME/.docker/config.json" > "$HOME/.docker/config.json.tmp"
        mv "$HOME/.docker/config.json.tmp" "$HOME/.docker/config.json"
   ```
- publish image to ACR using Microsoft.NET.Build.Containers
   ```yaml
    - name: publish
      run: dotnet publish ./my/path/to.csproj --os linux --arch x64 /t:PublishContainer -c Release
   ```

### don't use `az acr login` and log in directly to docker with a clientId and clientSecret (undesired security stature)
### don't use Microsoft.NET.Build.Containers and generate the Dockerfile in source (not PROGRESSING) 
