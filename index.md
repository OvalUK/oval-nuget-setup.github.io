# Creating a private Maven repository #

Sign in to the company GitHub account, create a new private repository for the package. It is not possible to create a maven package without a git repository.

![new repo](https://s3.eu-west-1.amazonaws.com/ovalgeneric/public/assets/nuget-guide/create-repo.png)

This will create your repository, you can push your package as usual

![repo](https://s3.eu-west-1.amazonaws.com/ovalgeneric/public/assets/nuget-guide/new-repo.png)

Take a note of the repository url, this will be required later when setting up you local authorisation.

---

## If you do not have an existing API key for your github account then you will need to create one ##

First go to settings > Developer settings

![repo](https://s3.eu-west-1.amazonaws.com/ovalgeneric/public/assets/nuget-guide/api-gen-1.png)

Then Personal access tokens > Generate new token

![repo](https://s3.eu-west-1.amazonaws.com/ovalgeneric/public/assets/nuget-guide/api-gen-2.png)

Click ```write:packages``` which will provide the necessary default privileges, add a Note as a reminder of what the token is being used for.

![repo](https://s3.eu-west-1.amazonaws.com/ovalgeneric/public/assets/nuget-guide/api-gen-3.png)

After the token generates copy it for use later.

---

## Setting up a package to push to your github repository ##

In your root project folder create a ```nuget.config``` file. Here we will add out Github credentials

In the _packageSources_ configuration add a url value to a nuget index file that will be generated in the organisation folder on GitHub. In the _packageSourceCredentials_ add a _github_ property with keys-values for you username and the API key

```XML
<?xml version="1.0" encoding="utf-8"?>
<configuration>
    <packageSources>
        <clear />
        <add key="github" value="https://nuget.pkg.github.com/OvalUK/index.json" />
    </packageSources>
    <packageSourceCredentials>
        <github>
            <add key="Username" value="alexwhiteoval" />
            <add key="ClearTextPassword" value="-- YOUR API KEY GOES HERE ---" />
        </github>
    </packageSourceCredentials>
</configuration>
```

In the ```.csproj``` you need to add some additional properties including the _RepositoryUrl_ which helps dotnet determine where to send the package.

```XML
<Project Sdk="Microsoft.NET.Sdk">

    <PropertyGroup>
        <TargetFramework>netstandard2.0</TargetFramework>
        <GeneratePackageOnBuild>true</GeneratePackageOnBuild>
        <PackageId>Oval.NugetTest</PackageId>
        <Version>1.0.0</Version>
        <Authors>Alex White</Authors>
        <Company>Oval Business Solutions</Company>
        <Product>Testing Resource</Product>
        <AssemblyName>Oval.NugetTest</AssemblyName>
        <RootNamespace>Oval.NugetTest</RootNamespace>
        <RepositoryUrl>https://github.com/OvalUK/nugettest.git</RepositoryUrl>
    </PropertyGroup>

</Project>
```

With this setup we can ```git push ...``` our repository, ```dotnet publish -c Release``` and ```dotnet nuget pack -c Release``` to publish the package to our private repository. In theory the pack command should read the API key from the config file but that has not been the case for me, if the same if true for you then add ```--api-key YOUR_API_KEY``` to the pack command.

---

## Pulling the package into a project ##

To pull a package into a new or existing project you will need to replicate the ```nuget.config``` above and update your ```.csproj``` and add the _PackageReference_ to an _ItemGroup_ property. This repository does not need the _RepositoryUrl_ property.

```XML
<Project Sdk="Microsoft.NET.Sdk">

  <PropertyGroup>
    <OutputType>Exe</OutputType>
    <TargetFramework>net5.0</TargetFramework>
    <PackageId>Oval.NugetTestPulling</PackageId>
    <Version>1.0.0</Version>
    <Authors>Alex White</Authors>
    <Company>Oval Business Solutions</Company>
    <Product>Testing Resource</Product>
    <AssemblyName>Oval.NugetTestPulling</AssemblyName>
    <RootNamespace>Oval.NugetTestPulling</RootNamespace>
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="Oval.NugetTest" Version="1.0.0" />
  </ItemGroup>
</Project>
```
