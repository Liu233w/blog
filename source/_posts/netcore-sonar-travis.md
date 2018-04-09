---
layout: blog
date: 2018-04-10 00:14:33
tags:
- .Net Core
- Travis-CI
- SonarCloud
title: 使用 Travis 和 SonarCloud(SonarAnalyzer) 对 .Net Core 项目进行分析
---

最近 SonarC# 加上了对 .Net Core 的支持，而且可以在 Linux 上运行 C# 的分析了。这样的话，就可以用 Travis-CI 对代码进行持续分析，并且自动将结果上传到 SonarCloud 上。要启用这个功能，只需要在 `.travis.yml` 里面加上 SonarCloud Addon（如果你用的话），然后在 install 字段里面下载sonar-scanner-msbuild的压缩包并运行即可。代码：

```yaml
  addons:
    sonarcloud:
      organization: "liu233w-github"
      token: $SONAR_TOKEN
  cache:
    directories:
      - $HOME/.sonar/cache
      - $HOME/.m2
  install:
    - curl -o scanner.zip -L https://github.com/SonarSource/sonar-scanner-msbuild/releases/download/4.1.1.1164/sonar-scanner-msbuild-4.1.1.1164-netcoreapp2.0.zip
    - unzip scanner.zip
    - chmod +x sonar-scanner-3.1.0.1141/bin/sonar-scanner
  script:
    - dotnet SonarScanner.MSBuild.dll begin /k:"acm-statistics-abp" /d:sonar.login=$SONAR_TOKEN /d:sonar.organization=liu233w-github /d:sonar.host.url=https://sonarcloud.io
    - dotnet build
    - dotnet SonarScanner.MSBuild.dll end /d:sonar.login=$SONAR_TOKEN
```

请将代码中的 project-key、organization-key 替换成自己的，然后在Travis设置里面的环境变量里加上 SONAR_TOKEN="在 SonarCloud 中的 Token" 即可。

这个配置文件里面会下载特定版本的sonar-scanner-msbuild并解压运行，注意里面的那句 `chmod +x sonar-scanner-3.1.0.1141/bin/sonar-scanner`，如果不加上的话dotnet core会抛出一个`System.ComponentModel.Win32Exception`，内容是`Native error= Access denied`，非常的迷，搞得我还以为这个版本还没有完全跨平台。

这段配置文件来自我的开源项目 [acm-statistics-abp](https://github.com/Liu233w/acm-statistics-abp/blob/master/.travis.yml)，下面是完整版的配置文件，分离了测试代码和 SonarCloud 分析代码。

```yaml
notifications:
  email:
    on_success: never
    on_failure: always

language: csharp
solution: AcmStatisticsAbp.sln

mono: none
dotnet: 2.0.0
dist: trusty

stages:
  - name: test
  - name: deploy
    if: tag =~ ^v
  - name: sonar

matrix:
  include:
    - env: NAME=Test
      stage: test
      script:
        - cd test/AcmStatisticsAbp.Tests/
        - dotnet test
    
    - env: NAME=Sonar-Analysis
      stage: sonar
      addons:
        sonarcloud:
          organization: "liu233w-github"
          token: $SONAR_TOKEN
      cache:
        directories:
          - $HOME/.sonar/cache
          - $HOME/.m2
      install:
        - curl -o scanner.zip -L https://github.com/SonarSource/sonar-scanner-msbuild/releases/download/4.1.1.1164/sonar-scanner-msbuild-4.1.1.1164-netcoreapp2.0.zip
        - unzip scanner.zip
        - chmod +x sonar-scanner-3.1.0.1141/bin/sonar-scanner
      script:
        - dotnet SonarScanner.MSBuild.dll begin /k:"acm-statistics-abp" /d:sonar.login=$SONAR_TOKEN /d:sonar.organization=liu233w-github /d:sonar.exclusions=**/wwwroot/**/* /d:sonar.host.url=https://sonarcloud.io
        - dotnet build
        - dotnet SonarScanner.MSBuild.dll end /d:sonar.login=$SONAR_TOKEN

```

