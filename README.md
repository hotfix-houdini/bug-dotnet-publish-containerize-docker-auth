# WIP
# Overview
It appears that dotnet publish directly to a docker registry using Microsoft.NET.Build.Containers __fails with an unauthorized error if you log into docker in certain situations (such as when an identity token is used)__

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

# Work arounds
