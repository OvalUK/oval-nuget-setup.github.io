# Creating a private Nuget repository

Sign in to the company based GitHub account, create a new private repository for the package. It is not possible to create a nuget package without a git repository.

![new repo](https://s3.eu-west-1.amazonaws.com/ovalgeneric/public/assets/nuget-guide/create-repo.png)

This will create your repository, you can push your package as usual

![repo](https://s3.eu-west-1.amazonaws.com/ovalgeneric/public/assets/nuget-guide/new-repo.png)

Take a note of the repository url, this will be required later when setting up you local authorisation.

---

## If you do not have an existing API key for your github account then you will need to create one

First go to settings > Developer settings

![repo](https://s3.eu-west-1.amazonaws.com/ovalgeneric/public/assets/nuget-guide/api-gen-1.png)

Then Personal access tokens > Generate new token

![repo](https://s3.eu-west-1.amazonaws.com/ovalgeneric/public/assets/nuget-guide/api-gen-2.png)

Click ```write:packages``` which will provide the necessary default privileges, add a Note as a reminder of what the token is being used for.

![repo](https://s3.eu-west-1.amazonaws.com/ovalgeneric/public/assets/nuget-guide/api-gen-3.png)

After the token generates copy it for use later.

---

## Setting up a package to push to your github repository

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

With this setup we can ```git push ...``` our repository, ```dotnet publish -c Release``` and ```dotnet nuget pack -c Release``` and finally to publish to our private repository ```dotnet nuget push PATH_TO_nupkg_FILE --source "github"```. In theory the ```nuget push``` command should read the API key from the config file but that has not been the case for me, if the same if true for you then add ```--api-key YOUR_API_KEY``` to the ```nuget push``` command.

### Automatic deployments with GitHub

If you using GitHub actions to complete the build and deployment process, the source repository needs to have a actions yaml file, name the file appropriately in the following location ```./.github/workflows/MY_ACTION.yml``` The contents should be as follows:

```YAML
name: Package Publisher

on:
  push:
    tags:
      - "*"
  release:
    types:
      - published
    
env:
  PROJECT_DIR: '.'
  PROJECT_NAME: NugetTest

  GITHUB_FEED: https://nuget.pkg.github.com/OvalUK/
  GITHUB_USER: ${{ github.actor }}
  GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}

jobs:
  build-and-deploy:
    # if: github.event_name == 'release'
    name: Build and Deploy
    runs-on: ubuntu-20.04
    environment: production
    steps:
    - uses: actions/checkout@v2
    - run: dotnet clean -c Release && dotnet nuget locals all --clear && dotnet publish -c Release
    - run: |
        arrTag=(${GITHUB_REF//\// })
        VERSION="${arrTag[2]}"
        VERSION="${VERSION//v}"
        dotnet pack -c Release --no-restore --include-symbols --include-source -p:PackageVersion=$VERSION $PROJECT_DIR/$PROJECT_NAME.csproj
    - name: Add GRP Source
      run: dotnet nuget add source https://nuget.pkg.github.com/ovaluk/index.json -n "GRP" -u $GITHUB_USER -p $GITHUB_TOKEN --store-password-in-clear-text
    - name: Publish package to repo
      run: |
        for f in "./bin/Release/*$VERSION.nupkg"
        do
          dotnet nuget push $f --source "GRP" --skip-duplicate --api-key $GITHUB_TOKEN
        done
```

This above action will build the package using the ubuntu-20.04 operating system with .NET5 installed. During the pack phase the version number will be updated using the tag value. Git tags should be in the format _v#.#.#_ or _v#.#.#-rc_ this will result in a dependency similar to:

```sh
dotnet add PROJECT package Oval.NugetTest --version 0.22.0-rc
```

The Add GRP and publish phases take each versioned package and pushes them to the private repository, including the associated symbols packages.

---

## Pulling the package into a project

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
