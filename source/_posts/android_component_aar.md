---
title: Android 组件化-组件aar化实战
date: 2021-07-31 14:16:55
cover: true
tags: 
    - 组件化
    - Maven
    - aar化
    - 编译优化
category: 
	- 组件化
summary: Gradle 推送插件、阿里云效 Maven 仓库、Gradle 依赖管理


---

![](https://raw.githubusercontent.com/jaydroid1024/jay_image_repo/main/img/post_banner_jay.png)



# Android 组件化-组件aar化实战

[toc]



## 1.Gradle 推送插件|Maven Publish Plugin

Maven Publish 插件提供将构建产物发布到[Apache Maven](http://maven.apache.org/)存储库的功能。发布到 Maven 存储库的模块可以被 Maven、Gradle和其他了解 Maven 存储库格式的工具使用。

### 1.1 插件任务|Tasks

- `generatePomFileFor*PubName*Publication`—[生成MavenPom](https://docs.gradle.org/7.0/dsl/org.gradle.api.publish.maven.tasks.GenerateMavenPom.html)

  为名为*PubName*的发布创建 POM 文件，填充已知元数据，例如项目名称、项目版本和依赖项。POM 文件的默认位置是*build/publications/$pubName/pom-default.xml*。

- `publish*PubName*PublicationTo*RepoName*Repository`— [PublishToMavenRepository](https://docs.gradle.org/7.0/dsl/org.gradle.api.publish.maven.tasks.PublishToMavenRepository.html)

  将*PubName*发布发布到名为*RepoName*的存储库。如果您有一个没有明确名称的存储库定义，*RepoName*将是“Maven”。

- `publish*PubName*PublicationToMavenLocal`— [PublishToMavenLocal](https://docs.gradle.org/7.0/javadoc/org/gradle/api/publish/maven/tasks/PublishToMavenLocal.html)

  将*PubName*发布与发布的 POM 文件和其他元数据一起复制到本地 Maven 缓存——通常是*$USER_HOME/.m2/repository*。

- `publish`

  *取决于*：所有任务`publish*PubName*PublicationTo*RepoName*Repository`将所有定义的发布发布到所有定义的存储库的聚合任务。它*不*包括复制出版物本地Maven缓存。

- `publishToMavenLocal`

  *取决于*：所有任务`publish*PubName*PublicationToMavenLocal`将所有定义的发布复制到本地 Maven 缓存，包括它们的元数据（POM 文件等）。

```groovy
//生成MavenPom
generateMetadataFileForDebugPublication
generateMetadataFileForReleasePublication
generatePomFileForDebugPublication 
generatePomFileForReleasePublication

//PublishToMavenRepository
publish
publishAllPublicationsToMavenRepository
publishDebugPublicationToMavenRepository
publishReleasePublicationToMavenRepository

//PublishToMavenLocal
publishToMavenLocal
publishDebugPublicationToMavenLocal
publishReleasePublicationToMavenLocal

//如果你的项目使用了gralde wrapper组件的话请使用以下命令
./gradlew task [name]
./gradlew task :lib-net:publishToMavenLocal
```



### 1.2 构建产物|Publications

您可以在 Maven 出版物中配置四项主要内容：

- A [component](https://docs.gradle.org/7.0/userguide/dependency_management_terminology.html#sub:terminology_component)  例如 一个 java module、Android library module
  - 通过该方法指定 MavenPublication.from(org.gradle.api.component.SoftwareComponent)
- [Custom artifacts](https://docs.gradle.org/7.0/userguide/publishing_customization.html#sec:publishing_custom_artifacts_to_maven) 
  - 自定义构建产物通过这个方法 [MavenPublication.artifact(java.lang.Object)](https://docs.gradle.org/7.0/dsl/org.gradle.api.publish.maven.MavenPublication.html#org.gradle.api.publish.maven.MavenPublication:artifact(java.lang.Object)) method.
- 标准的元数据 
  - 例如 artifactId、groupId、version.
- POM file 的其它配置
  - 通过这个方法设置 MavenPublication.pom(org.gradle.api.Action)



### 1.3 仓库|Repositories

插件提供 MavenArtifactRepository 类型的存储库

定义发布存储库：

```groovy
publishing {
   repositories {
      maven {
          url "url"
          credentials {
              username = 'name'
              password = 'pwd'
          }
      }
		}
}
```

Snapshot and release repositories

```groovy
publishing {
    repositories {
        maven {
            def releasesRepoUrl = layout.buildDirectory.dir('repos/releases')
            def snapshotsRepoUrl = layout.buildDirectory.dir('repos/snapshots')
            url = version.endsWith('SNAPSHOT') ? snapshotsRepoUrl : releasesRepoUrl
        }
    }
}
```



### 1.4 完整示例 

以下示例演示了如何签署和发布包含源代码、Javadoc 和自定义 POM 的 Java 库：

build.gradle

```groovy
plugins {
    id 'java-library'
    id 'maven-publish'
    id 'signing'
}

group = 'com.example'
version = '1.0'

java {
    withJavadocJar()
    withSourcesJar()
}

publishing {
    publications {
        mavenJava(MavenPublication) {
            artifactId = 'my-library'
            from components.java
            versionMapping {
                usage('java-api') {
                    fromResolutionOf('runtimeClasspath')
                }
                usage('java-runtime') {
                    fromResolutionResult()
                }
            }
            pom {
                name = 'My Library'
                description = 'A concise description of my library'
                url = 'http://www.example.com/library'
                properties = [
                    myProp: "value",
                    "prop.with.dots": "anotherValue"
                ]
                licenses {
                    license {
                        name = 'The Apache License, Version 2.0'
                        url = 'http://www.apache.org/licenses/LICENSE-2.0.txt'
                    }
                }
                developers {
                    developer {
                        id = 'johnd'
                        name = 'John Doe'
                        email = 'john.doe@example.com'
                    }
                }
                scm {
                    connection = 'scm:git:git://example.com/my-library.git'
                    developerConnection = 'scm:git:ssh://example.com/my-library.git'
                    url = 'http://example.com/my-library/'
                }
            }
        }
    }
    repositories {
        maven {
            // change URLs to point to your repos, e.g. http://my.org/repo
            def releasesRepoUrl = layout.buildDirectory.dir('repos/releases')
            def snapshotsRepoUrl = layout.buildDirectory.dir('repos/snapshots')
            url = version.endsWith('SNAPSHOT') ? snapshotsRepoUrl : releasesRepoUrl
            credentials {
              username = 'name'
              password = 'pwd'
            }
        }
    }
}

signing {
    sign publishing.publications.mavenJava
}


javadoc {
    if(JavaVersion.current().isJava9Compatible()) {
        options.addBooleanOption('html5', true)
    }
}
```

以上配置的结果是将发布以下工件：

- POM文件: `my-library-1.0.pom`
- 主要的 JAR 工件 : `my-library-1.0.jar`
- 已显式配置的源代码: `my-library-1.0-sources.jar`
- 已显式配置的 Javadoc： `my-library-1.0-javadoc.jar`

签名插件用于为每个工件生成签名文件。此外，将为所有工件和签名文件生成校验和文件。



### 1.5 Android 中 使用 Maven 发布插件

Android Gradle 插件 3.6.0 及更高版本包括对 [Maven Publish Gradle 插件的支持](https://docs.gradle.org/current/userguide/publishing_maven.html)，它允许您将构建工件发布到 Apache Maven 存储库。Android Gradle 插件 为您的应用程序或库模块中的每个构建变体工件创建一个 [*组件*](https://docs.gradle.org/current/userguide/dependency_management_terminology.html#sub:terminology_component)，您可以使用它来自定义 [*发布*](https://docs.gradle.org/current/userguide/publishing_maven.html#publishing_maven:publications) 到 Maven 存储库。

Android 插件创建的组件取决于模块是使用应用程序插件还是库插件，如下表所述。

| Android Gradle 插件       | 出版神器                                       | 组件名称                 |
| :------------------------ | :--------------------------------------------- | :----------------------- |
| `com.android.library`     | AAR                                            | `components.variant`     |
| `com.android.application` | APK 的 ZIP，以及可用的 ProGuard 或 R8 映射文件 | `components.variant_apk` |
| `com.android.application` | 一个 Android 应用程序包 (AAB)                  | `components.variant_aab` |



### 1.6 Android 版示例

```groovy
// Because the components are created only during the afterEvaluate phase, you must
// configure your publications using the afterEvaluate() lifecycle method.
afterEvaluate {
    publishing {
        publications {
            // Creates a Maven publication called "release".
            release(MavenPublication) {
                // Applies the component for the release build variant.
                from components.release
                // You can then customize attributes of the publication as shown below.
                groupId = GROUP
                artifactId = ARTIFACT_ID
                version = VERSION

            }
            // Creates a Maven publication called “debug”.
            debug(MavenPublication) {
                // Applies the component for the debug build variant.
                from components.debug
                groupId = GROUP
                artifactId = ARTIFACT_ID + "-debug"
                version = VERSION
            }
        }
        repositories {
            maven {
                // change URLs to point to your repos, e.g. http://my.org/repo
                url = URL
                credentials {
                    username USER_NAME
                    password PASSWORD
                }
            }
        }
    }
}



```

### 1.7 Android 自定义POM文件版示例

```groovy
apply plugin: 'maven-publish'

println("--------${project.name}：Maven Publish Gradle--------")
//release 和 snapshot 的控制开关
def isUploadToRelease = rootProject.ext.mavenRepo['isUploadToRelease']
//远程Maven仓库的URL Release
def MAVEN_REPO_RELEASE_URL = rootProject.ext.mavenRepo['mavenRepoUrlRelease']
//远程Maven仓库的URL snapshots
def MAVEN_REPO_SNAPSHOTS_URL = rootProject.ext.mavenRepo['mavenRepoUrlSnapshots']
//远程Maven仓库用户名
def USER_NAME = rootProject.ext.mavenRepo['userName']
//远程Maven仓库密码
def PASSWORD = rootProject.ext.mavenRepo['password']
// 唯一标识 每个组件都要指定
def GROUP = group.toString()
// todo 默认为项目名称
def ARTIFACT_ID = project.name
// 版本号 每个组件都要指定
def VERSION = version.toString()
//远程Maven仓库的URL
def URL = isUploadToRelease ? MAVEN_REPO_RELEASE_URL : MAVEN_REPO_SNAPSHOTS_URL

println("dependencies_path: $GROUP:$ARTIFACT_ID:$VERSION")
println("MAVEN_REPO_RELEASE_URL: $MAVEN_REPO_RELEASE_URL")
println("MAVEN_REPO_SNAPSHOTS_URL: $MAVEN_REPO_SNAPSHOTS_URL")


//https://docs.gradle.org/7.0/userguide/publishing_maven.html#publishing_maven
//https://developer.android.com/studio/build/maven-publish-plugin

// Because the components are created only during the afterEvaluate phase, you must
// configure your publications using the afterEvaluate() lifecycle method.


task sourceJar(type: Jar) {
    from android.sourceSets.main.java.srcDirs
    classifier "sources"
}

afterEvaluate {
    publishing {
        publications {
            maven(MavenPublication) {
                groupId GROUP
                artifactId ARTIFACT_ID
                version VERSION
                artifact bundleReleaseAar
                artifact sourceJar

                //根据输入数据生成 POM 后，自定义配置 POM。
                pom.withXml {
                 
                    final dependenciesNode = asNode().appendNode('dependencies')

                    //dependenciesNode:dependencies[attributes={}; value=[]]
                    println "dependenciesNode:" + dependenciesNode
                    ext.addDependency = { Dependency dep, String scope ->
                        //Dependency:DefaultExternalModuleDependency{group='com.qlife.android', name='lib-baidu-face', version='1.0.0', configuration='default'}
                        //scope:compile
                        println "Dependency:" + dep
                        println "scope:" + scope

                        if (dep.group == null || dep.version == null || dep.name == null || dep.name == "unspecified")
                            return // invalid dependencies should be ignored

                        final dependencyNode = dependenciesNode.appendNode('dependency')
                        dependencyNode.appendNode('artifactId', dep.name)

                        if (dep.version == 'unspecified') {
                            dependencyNode.appendNode('groupId', project.ext.pomGroupID)
                            dependencyNode.appendNode('version', project.ext.pomVersion)
                            System.println("${project.ext.pomGroupID} ${dep.name} ${project.ext.pomVersion}")
                        } else {
                            dependencyNode.appendNode('groupId', dep.group)
                            dependencyNode.appendNode('version', dep.version)
                            System.println("${dep.group} ${dep.name} ${dep.version}")
                        }

                        dependencyNode.appendNode('scope', scope)
                        //一些依赖可能有类型，比如aar，应该在POM文件中提到
                        // Some dependencies may have types, such as aar, that should be mentioned in the POM file
                        def artifactsList = dep.properties['artifacts']
                        if (artifactsList != null && artifactsList.size() > 0) {
                            final artifact = artifactsList[0]
                            dependencyNode.appendNode('type', artifact.getType())
                        }

                        if (!dep.transitive) {
                            //在非传递依赖的情况下，它的所有依赖都应该从 POM 文件中强制排除
                            // In case of non transitive dependency, all its dependencies should be force excluded from them POM file
                            final exclusionNode = dependencyNode.appendNode('exclusions').appendNode('exclusion')
                            exclusionNode.appendNode('groupId', '*')
                            exclusionNode.appendNode('artifactId', '*')
                        } else if (!dep.properties.excludeRules.empty) {
                            //对于带排除的传递，应将所有排除规则添加到 POM 文件中
                            // For transitive with exclusions, all exclude rules should be added to the POM file
                            final exclusions = dependencyNode.appendNode('exclusions')
                            dep.properties.excludeRules.each { ExcludeRule rule ->
                                final exclusionNode = exclusions.appendNode('exclusion')
                                exclusionNode.appendNode('groupId', rule.group ?: '*')
                                exclusionNode.appendNode('artifactId', rule.module ?: '*')
                            }
                        }
                    }

                    // List all "api" dependencies (for new Gradle) as "compile" dependencies
                    configurations.api.getDependencies().each { dep -> addDependency(dep, "compile") }
                    // List all "implementation" dependencies (for new Gradle) as "runtime" dependencies
                    configurations.implementation.getDependencies().each { dep -> addDependency(dep, "runtime") }

                }
            }
        }
        repositories {
            maven {
                url = URL
                credentials {
                    username USER_NAME
                    password PASSWORD
                }
            }
        }
    }
}

task cleanBuildPublishLocal(type: GradleBuild) {
    tasks = ['clean', 'build', 'publishToMavenLocal']
}

task cleanBuildPublish(type: GradleBuild) {
    tasks = ['clean', 'build', 'publish']
}



```



### 1.8 Maven pom 文件格式

POM( Project Object Model，项目对象模型 ) 是 Maven 工程的基本工作单元，是一个XML文件，包含了项目的基本信息，用于描述项目如何构建，声明项目依赖，等等。

执行任务或目标时，Maven 会在当前目录中查找 POM。它读取 POM，获取所需的配置信息，然后执行目标。

POM 中可以指定以下配置：

- 项目依赖
- 插件
- 执行目标
- 项目构建 profile
- 项目版本
- 项目开发者列表
- 相关邮件列表信息

所有 POM 文件都需要 project 元素和三个必需字段：groupId，artifactId，version。

| 节点         | 描述                                                         |
| :----------- | :----------------------------------------------------------- |
| project      | 工程的根标签。                                               |
| modelVersion | 模型版本需要设置为 4.0.0。                                   |
| groupId      | 公司或者组织的唯一标志，并且配置时生成的路径也是由此生成， 如com.companyname.project-group，maven会将该项目打成的jar包放本地路径：/com/companyname/project-group |
| artifactId   | 项目的唯一ID，一个groupId下面可能多个项目，就是靠artifactId来区分的。groupId 和 artifactId 一起定义了 artifact 在仓库中的位置。 |
| version      | 这是工程的版本号。在 artifact 的仓库中，它用来区分不同的版本。 |
| packaging    | 项目产生的构件类型，例如jar、war、ear、pom。插件可以创建他们自己的构件类型，所以前面列的不是全部构件类型 |
| dependencies | 该元素描述了项目相关的所有依赖。 这些依赖组成了项目构建过程中的一个个环节。它们自动从项目定义的仓库中下载。 |
| dependency   | 依赖项                                                       |
| scope        | 依赖范围。在项目发布过程中，帮助决定哪些构件被包括进来。欲知详情请参考依赖机制。 <br />- compile ：默认范围，用于编译<br /> - provided：类似于编译，但支持你期待jdk或者容器提供，类似于classpath   <br />- runtime: 在执行时需要使用 <br />- test: 用于test任务时使用 <br />- system: 需要外在提供相应的元素。通过systemPath来取得                 <br />- systemPath: 仅用于范围为system。提供相应的路径 <br />- optional: 当项目自身被依赖时，标注依赖是否传递。用于连续依赖时使用 |

[POM 标签大全详解](https://www.runoob.com/maven/maven-pom.html)

一个简单的Android 依赖库的pom文件如下

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd" xmlns="http://maven.apache.org/POM/4.0.0"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">
  <!-- This module was also published with a richer model, Gradle metadata,  -->
  <!-- which should be used instead. Do not delete the following line which  -->
  <!-- is to indicate to Gradle or any Gradle module metadata file consumer  -->
  <!-- that they should prefer consuming it instead. -->
  <!-- do_not_remove: published-with-gradle-metadata -->
  <modelVersion>4.0.0</modelVersion>
  <groupId>com.qlife.android</groupId>
  <artifactId>lib-net-release</artifactId>
  <version>1.0.1</version>
  <packaging>aar</packaging>
  
  <dependencies>
    <dependency>
      <groupId>com.squareup.retrofit2</groupId>
      <artifactId>retrofit</artifactId>
      <version>2.6.0</version>
      <scope>compile</scope>
    </dependency>
    <dependency>
      <groupId>com.squareup.okhttp3</groupId>
      <artifactId>logging-interceptor</artifactId>
      <version>4.2.2</version>
      <scope>runtime</scope>
    </dependency>
    <dependency>
      <groupId>com.github.jaydroid1024.JDispatcher</groupId>
      <artifactId>jdispatcher-api</artifactId>
      <version>0.0.7</version>
      <scope>runtime</scope>
    </dependency>
    。。。
  </dependencies>
</project>

```





## 2.阿里云-云效 Maven 仓库

如果想自己搭建 NXRM(Nexus Repository Manager)私服，可参考 [Nexus 官网]( https://help.sonatype.com/repomanager3) [下载NXRM](https://help.sonatype.com/repomanager3/download)

阿里云Maven中央仓库为 [阿里云云效](https://devops.aliyun.com/?channel=maven.aliyun) 提供的公共代理仓库，帮助研发人员提高研发生产效率，使用阿里云Maven中央仓库作为下载源，速度更快更稳定。

[阿里云云效](https://devops.aliyun.com/?channel=maven.aliyun) 是企业级一站式 DevOps 平台，覆盖产品从需求到运营的研发全生命周期，其中云效也提供了免费、可靠的Maven私有仓库 [Packages](https://packages.aliyun.com/?channel=maven.aliyun)



### 2.1 公共代理仓库

| 仓库名称         | 阿里云仓库地址                                       | 阿里云仓库地址(老版)                                         | 源地址                                   |
| :--------------- | :--------------------------------------------------- | :----------------------------------------------------------- | :--------------------------------------- |
| central          | https://maven.aliyun.com/repository/central          | https://maven.aliyun.com/nexus/content/repositories/central  | https://repo1.maven.org/maven2/          |
| jcenter          | https://maven.aliyun.com/repository/public           | https://maven.aliyun.com/nexus/content/repositories/jcenter  | http://jcenter.bintray.com/              |
| public           | https://maven.aliyun.com/repository/public           | https://maven.aliyun.com/nexus/content/groups/public         | central仓和jcenter仓的聚合仓             |
| google           | https://maven.aliyun.com/repository/google           | https://maven.aliyun.com/nexus/content/repositories/google   | https://maven.google.com/                |
| gradle-plugin    | https://maven.aliyun.com/repository/gradle-plugin    | https://maven.aliyun.com/nexus/content/repositories/gradle-plugin | https://plugins.gradle.org/m2/           |
| spring           | https://maven.aliyun.com/repository/spring           | https://maven.aliyun.com/nexus/content/repositories/spring   | http://repo.spring.io/libs-milestone/    |
| spring-plugin    | https://maven.aliyun.com/repository/spring-plugin    | https://maven.aliyun.com/nexus/content/repositories/spring-plugin | http://repo.spring.io/plugins-release/   |
| grails-core      | https://maven.aliyun.com/repository/grails-core      | https://maven.aliyun.com/nexus/content/repositories/grails-core | https://repo.grails.org/grails/core      |
| apache snapshots | https://maven.aliyun.com/repository/apache-snapshots | https://maven.aliyun.com/nexus/content/repositories/apache-snapshots | https://repository.apache.org/snapshots/ |



### 2.2 Android 相关代理仓库

| 仓库名称      | 阿里云仓库地址                                    |
| ------------- | ------------------------------------------------- |
| central       | https://maven.aliyun.com/repository/central       |
| jcenter       | https://maven.aliyun.com/repository/public        |
| public        | https://maven.aliyun.com/repository/public        |
| google        | https://maven.aliyun.com/repository/google        |
| gradle-plugin | https://maven.aliyun.com/repository/gradle-plugin |

**拿来就用**

```groovy
buildscript {
    repositories {
      //central
      maven { url 'https://maven.aliyun.com/repository/central' }
      //jcenter&public
      maven { url 'https://maven.aliyun.com/repository/public' }
      //google
      maven { url 'https://maven.aliyun.com/repository/google' }
      //gradle-plugin
      maven { url 'https://maven.aliyun.com/repository/gradle-plugin' }
      mavenCentral()
      maven { url "https://jitpack.io" }
      google()
      jcenter()
    }
}
```



### 2.3 制品私有仓库

- 云效 Packages 为您自动创建了两个 Maven 仓库，一个 release 库和一个 snapshot 库。

  - Maven Release 库用于存储功能趋于稳定、当前更新停止，可以用于发行的版本。
  - Maven Snapchat 库用于存储不稳定、尚处于开发中的版本，即快照版本。

  - 您的制品文件具体推送到哪个库，根据您项目目录的build.gradle文件中version字段中是否配置了-SNAPSHOT。

- 进入仓库后，可以通过仓库指南完成 仓库凭证设置、制品文件的上传和下载、私有库迁移。

- 包列表下展示仓库下所有二进制包文件，支持通过 Group Id 和 Artifacts Id 进行包文件搜索。

- 点击包文件展示包文件信息，默认展示最新版本信息，点击可切换版本。

- 默认企业拥有者为仓库拥有者，其他企业成员需要在仓库中设置成员和角色。仓库公开性、成员角色之间的关系如下：

|            |                                  |                                  |
| ---------- | -------------------------------- | -------------------------------- |
| 仓库角色   | 仓库公开性：私有仓库             | 仓库公开性：企业内可见           |
| 拥有者     | 访问、下载、上传、删除、仓库管理 | 访问、下载、上传、删除、仓库管理 |
| 管理员     | 访问、下载、上传、删除、仓库管理 | 访问、下载、上传、删除、仓库管理 |
| 开发成员   | 访问、下载、上传                 | 访问、下载、上传                 |
| 普通成员   | 访问、下载                       | 访问、下载                       |
| 非仓库成员 | 无                               | 访问、下载                       |



### 2.4 Gradle 推送

1. 设置仓库凭证

```groovy
apply plugin: 'maven-publish'

println("--------${project.name}：Maven Publish Gradle--------")
//release 和 snapshot 的控制开关
def isUploadToRelease = rootProject.ext.mavenRepo['isUploadToRelease']
//远程Maven仓库的URL Release
def MAVEN_REPO_RELEASE_URL = rootProject.ext.mavenRepo['mavenRepoUrlRelease']
//远程Maven仓库的URL snapshots
def MAVEN_REPO_SNAPSHOTS_URL = rootProject.ext.mavenRepo['mavenRepoUrlSnapshots']
//远程Maven仓库用户名
def USER_NAME = rootProject.ext.mavenRepo['userName']
//远程Maven仓库密码
def PASSWORD = rootProject.ext.mavenRepo['password']
// 唯一标识 每个组件都要指定
def GROUP = group.toString()
// todo 默认为项目名称
def ARTIFACT_ID = project.name
// 版本号 每个组件都要指定
def VERSION = version.toString()
//远程Maven仓库的URL
def URL = isUploadToRelease ? MAVEN_REPO_RELEASE_URL : MAVEN_REPO_SNAPSHOTS_URL

println("dependencies_path: $GROUP:$ARTIFACT_ID:$VERSION")
println("MAVEN_REPO_RELEASE_URL: $MAVEN_REPO_RELEASE_URL")
println("MAVEN_REPO_SNAPSHOTS_URL: $MAVEN_REPO_SNAPSHOTS_URL")


//https://docs.gradle.org/7.0/userguide/publishing_maven.html#publishing_maven
//https://developer.android.com/studio/build/maven-publish-plugin
// Because the components are created only during the afterEvaluate phase, you must
// configure your publications using the afterEvaluate() lifecycle method.
afterEvaluate {
    publishing {
        publications {
            // Creates a Maven publication called "release".
            release(MavenPublication) {
                // Applies the component for the release build variant.
                from components.release
                // You can then customize attributes of the publication as shown below.
                groupId = GROUP
                artifactId = ARTIFACT_ID
                version = VERSION

            }
            // Creates a Maven publication called “debug”.
            debug(MavenPublication) {
                // Applies the component for the debug build variant.
                from components.debug
                groupId = GROUP
                artifactId = ARTIFACT_ID + "-debug"
                version = VERSION
            }
        }
        repositories {
            maven {
                url = URL
                credentials {
                    username USER_NAME
                    password PASSWORD
                }
            }
        }
    }
}
```

2. 设置仓库下载配置

```groovy
allprojects {
    repositories {
        //central
        maven { url 'https://maven.aliyun.com/repository/central' }
        //jcenter&public
        maven { url 'https://maven.aliyun.com/repository/public' }
        //google
        maven { url 'https://maven.aliyun.com/repository/google' }
        //gradle-plugin
        maven { url 'https://maven.aliyun.com/repository/gradle-plugin' }
        mavenCentral()
        maven { url "https://jitpack.io" }
        google()
        jcenter()

        /**Maven 私服配置*/
        // 仓库类型 local dev production
        def currentMavenRepositoryType = rootProject.ext.mavenRepo['currentMavenRepositoryType']
        def localMavenRepositoryType = rootProject.ext.mavenRepositoryType['local']
        maven {
            //区分本地仓库和远程仓库
            if (currentMavenRepositoryType != localMavenRepositoryType) {
                credentials {
                    username rootProject.ext.mavenRepo['userName']
                    password rootProject.ext.mavenRepo['password']
                }
            }
            url rootProject.ext.mavenRepo['mavenRepoUrlRelease']
        }
        maven {
            //区分本地仓库和远程仓库
            if (currentMavenRepositoryType != localMavenRepositoryType) {
                credentials {
                    username rootProject.ext.mavenRepo['userName']
                    password rootProject.ext.mavenRepo['password']
                }
            }
            url rootProject.ext.mavenRepo['mavenRepoUrlSnapshots']
        }

    }
}
```



### 2.5 Gradle 拉取

1. 设置仓库凭证

```groovy
allprojects {
    repositories {
        //central
        maven { url 'https://maven.aliyun.com/repository/central' }
        //jcenter&public
        maven { url 'https://maven.aliyun.com/repository/public' }
        //google
        maven { url 'https://maven.aliyun.com/repository/google' }
        //gradle-plugin
        maven { url 'https://maven.aliyun.com/repository/gradle-plugin' }
        mavenCentral()
        maven { url "https://jitpack.io" }
        google()
        jcenter()

        /**Maven 私服配置*/
        // 仓库类型 local dev production
        def currentMavenRepositoryType = rootProject.ext.mavenRepo['currentMavenRepositoryType']
        def localMavenRepositoryType = rootProject.ext.mavenRepositoryType['local']
        maven {
            //区分本地仓库和远程仓库
            if (currentMavenRepositoryType != localMavenRepositoryType) {
                credentials {
                    username rootProject.ext.mavenRepo['userName']
                    password rootProject.ext.mavenRepo['password']
                }
            }
            url rootProject.ext.mavenRepo['mavenRepoUrlRelease']
        }
        maven {
            //区分本地仓库和远程仓库
            if (currentMavenRepositoryType != localMavenRepositoryType) {
                credentials {
                    username rootProject.ext.mavenRepo['userName']
                    password rootProject.ext.mavenRepo['password']
                }
            }
            url rootProject.ext.mavenRepo['mavenRepoUrlSnapshots']
        }

    }
}
```

2. 配置包信息

```groovy
dependencies {
    def currentMavenRepositoryType = rootProject.ext.mavenRepo['dependenceTypeIsModule']
    if (currentMavenRepositoryType) {
        implementation project(path: ':lib-net')
    } else {
        implementation rootProject.ext.libNet
    }
}
```



## 3.Grade依赖管理

### 3.1 依赖项类型

```groovy
dependencies {
  
    // 对本地库模块的依赖
    implementation project(":mylibrary")

    // 对本地二进制文件的依赖
    implementation fileTree(dir: 'libs', include: ['*.jar'])
  	implementation files('libs/foo.jar')
    implementation project(path: ':foo-aar-module')
  
    // 对远程二进制文件的依赖
    implementation 'com.example.android:app-magic:12.3'

}
```

**本地库模块依赖**

这里依赖了一个名为“mylibrary”（此名称必须与在您的 settings.gradle 文件中使用 `include:` 定义的库名称相符）的Android 库模块。在构建您的应用时，构建系统会编译该库模块，并将生成的编译内容打包到 APK 中。

**本地二进制文件依赖**

Gradle 声明了对项目的 `module_name/libs/` 目录中 JAR 文件的依赖关系（因为 Gradle 会读取 `build.gradle` 文件的相对路径）。

也可指定各个jar/aar文件或者通过创建一个aar/jar 模块建立像对本地库模块一样的依赖

**远程二进制文件依赖**

```
implementation 'com.example.android:app-magic:12.3'
```

这实际上是以下代码的简写形式：

```
implementation group: 'com.example.android', name: 'app-magic', version: '12.3'
```

这声明了对“com.example.android”命名空间组内的 12.3 版“app-magic”库的依赖关系。

此类远程依赖项要求您声明适当的远程代码库，Gradle 应在其中查找相应的库。如果本地不存在相应的库，Gradle 会从远程站点提取它。

### 3.2 依赖项依赖方式配置

在 `dependencies` 代码块内，您可以从多种不同的依赖项配置中选择其一,每种依赖项配置都向 Gradle 提供了有关如何使用该依赖项的不同说明。下表介绍了Android 项目中的依赖项使用的各种配置。

|        新配置         | 已弃用配置 | 行为描述                                                     |
| :-------------------: | :--------: | :----------------------------------------------------------- |
|   `implementation`    | `compile`  | Gradle 会将依赖项添加到编译类路径，并将依赖项打包到构建输出。 当模块配置 `implementation` 依赖项时，Gradle在编译时**不会将该依赖项传递给其他模块**。也就是说，其他模块只有在运行时才能使用该依赖项。 使用此依赖项配置代替 `api` 或 `compile`（已弃用）可以**显著缩短构建时间**，因为这样可以减少构建系统需要重新编译的模块数。例如，如果 `implementation` 依赖项更改了其 API，Gradle 只会重新编译该依赖项以及直接依赖于它的模块。大多数应用和测试模块都应使用此配置。 |
|         `api`         | `compile`  | Gradle 会将依赖项添加到编译类路径和构建输出。当一个模块包含 `api` 依赖项时，会让 Gradle **以传递方式将该依赖项导出到其他模块**，以便这些模块在运行时和编译时都可以使用该依赖项。 此配置的行为类似于 `compile`（现已弃用），但使用它时应格外小心，只能对需要以传递方式导出到其它上游消费者的依赖项时才使用它。这是因为，如果 `api` 依赖项更改了其外部 API，Gradle 会在编译时重新编译所有有权访问该依赖项的模块。因此，拥有大量的 `api` 依赖项会显**著增加构建时间**。除非要将依赖项的 API 公开给单独的模块，否则库模块应改用 `implementation` 依赖项。 |
|     `compileOnly`     | `provided` | Gradle 只会将依赖项添加到编译类路径也就是说，**不会将其添加到构建输出**。如果您创建 Android 模块时在编译期间需要相应依赖项，但它在运行时可有可无，此配置会很有用。如果您使用此配置，那么您的库模块必须包含一个运行时条件，用于检查是否提供了相应依赖项，然后适当地改变该模块的行为，以使该模块在未提供相应依赖项的情况下仍可正常运行。这样做**不会添加不重要的瞬时依赖项**，因而**有助于减小最终 APK 的大小**。此配置的行为类似于 `provided`（现已弃用）。**注意**：您不能将 `compileOnly` 配置与 AAR 依赖项配合使用。 |
|     `runtimeOnly`     |   `apk`    | Gradle 只会将依赖项添加到构建输出，以便在运行时使用。也就是说，不会将其添加到编译类路径。此配置的行为类似于 `apk`（现已弃用）。 |
| `annotationProcessor` | `compile`  | 如需添加对作为注释处理器的库的依赖关系，您必须使用 `annotationProcessor` 配置将其添加到注释处理器类路径。这是因为，使用此配置可以**将编译类路径与注释处理器类路径分开，从而提高构建性能**。如果 Gradle 在编译类路径上找到注释处理器，则会禁用[避免编译](https://docs.gradle.org/current/userguide/java_plugin.html#sec:java_compile_avoidance)功能，这样会对构建时间产生负面影响（Gradle 5.0 及更高版本会忽略在编译类路径上找到的注释处理器）。如果 JAR 文件包含以下文件，则 Android Gradle 插件会假定依赖项是注释处理器： `META-INF/services/javax.annotation.processing.Processor`。如果插件检测到编译类路径上包含注释处理器，则会生成构建错误。 |



### 3.3 依赖项顺序

依赖项的列出顺序指明了每个库的优先级：第一个库的优先级高于第二个，第二个库的优先级高于第三个，依此类推。在[合并资源](https://developer.android.com/studio/write/add-resources#resource_merging)或[将清单元素从库中合并](https://developer.android.com/studio/build/manifest-merge)到应用中时，此顺序很重要。

例如，如果您的项目声明以下内容：

- 依赖 `LIB_A` 和 `LIB_B`（按此顺序）
- `LIB_A` 依赖于 `LIB_C` 和 `LIB_D`（按此顺序）
- `LIB_B` 也依赖于 `LIB_C`

```groovy
//app
dependencies {
    implementation('LIB_A')
    implementation('LIB_B')
}
//LIB_A
dependencies {
    implementation('LIB_C')
    implementation('LIB_D')
}
//LIB_B
dependencies {
    implementation('LIB_C')
}
```

![image-20210826160438305](/Users/xuejiewang/Library/Application Support/typora-user-images/image-20210826160438305.png)

那么，扁平型依赖项顺序将如下所示：

1. `LIB_A`
2. `LIB_D`
3. `LIB_B`
4. `LIB_C`

这可以确保 `LIB_A` 和 `LIB_B` 都可以替换 `LIB_C`；并且 `LIB_D` 的优先级仍高于 `LIB_B`，因为 `LIB_A`（依赖前者）的优先级高于 `LIB_B`。

### 3.4 依赖冲突问题

**解决类路径之间的冲突**

当 Gradle 解析编译类路径时，会先解析运行时类路径，然后使用所得结果确定应添加到编译类路径的依赖项版本。换句话说，运行时类路径决定了下游类路径上完全相同的依赖项所需的版本号。

应用的运行时类路径还决定了 Gradle 需要对应用的测试 APK 的运行时类路径中的匹配依赖项使用的版本号。图 1 说明了类路径的层次结构。

![img](https://developer.android.com/studio/images/build/classpath_sync-2x.png)

例如，当应用使用 `implementation` [依赖项配置](https://developer.android.com/studio/build/dependencies#dependency_configurations)加入某个依赖项的一个版本，而库模块使用 `runtimeOnly` 配置加入该依赖项的另一个版本时，就可能会发生多个类路径中出现同一依赖项的不同版本的冲突。

在解析对运行时和编译时类路径的依赖关系时，Android Gradle 插件 3.3.0 及更高版本会尝试自动解决某些下游版本冲突。例如，如果运行时类路径包含库 A 版本 2.0，而编译类路径包含库 A 版本 1.0，则插件会自动将对编译类路径的依赖关系更新为库 A 版本 2.0，以避免错误。

不过，如果运行时类路径包含库 A 版本 1.0，而编译类路径包含库 A 版本 2.0，插件不会将对编译类路径的依赖关系降级为库 A 版本 1.0，您仍会收到一条与以下内容类似的错误：

```
Conflict with dependency 'com.example.library:some-lib:2.0' in project 'my-library'.
Resolved versions for runtime classpath (1.0) and compile classpath (2.0) differ.
```

如需解决此问题，请执行以下某项操作：

- 将所需版本的依赖项作为 `api` 依赖项添加到库模块。也就是说，只有库模块声明相应依赖项，但应用模块也能以传递方式访问其 API。 - 或者，您也可以同时在两个模块中声明相应依赖项，但应确保每个模块使用的版本相同。不妨考虑[配置项目全局属性](https://developer.android.com/studio/build/gradle-tips#configure-project-wide-properties)，以确保每个依赖项的版本在整个项目中保持一致。

**排除传递依赖项**

随着应用的范围不断扩大，它可能会包含许多依赖项，包括直接依赖项和传递依赖项（应用中导入的库所依赖的库）。如需排除不再需要的传递依赖项，您可以使用 `exclude` 关键字，如下所示：

```
dependencies {
    implementation('some-library') {
        exclude group: 'com.example.imgtools', module: 'native'
    }
}
```

如果在configuration中定义一个exclude,那么所有依赖的transitive dependency (指定的)都会被去除。定义exclude时候，或只指定group, 或只指定module名字，或二者都指定。

下面是一些使用exclude的典型场合：

- 有licensing问题
- 从远程仓库上无法获取到依赖
- runtime时候用不到
- 有版本冲突

**强制使用当前版本**

```
implementation('com.squareup.okhttp:okhttp-mt:2.5.0') {
    force = true
}
```

如上，我们在依赖okhttp的时候很可能发生冲突，就比如依赖的依赖中也包含了okhttp，这种场合下，就会产生版本冲突的问题，加上force = true表明的意思就是即使在有依赖库版本冲突的情况下坚持使用被标注的这个依赖库版本。

**间接依赖 transitive**

transitive dependencies 被称为依赖的依赖，称为“间接依赖”比较合适。

```
implementation('com.meituan.android.terminus:library:6.6.1.16@aar') {
    transitive = true
}
```

在后面加上@aar，意指你只是下载该aar包，而并不下载该aar包所依赖的其他库，那如果想在使用@aar的前提下还能下载其依赖库，则需要添加transitive=true的条件。



## 4.参考资料

- [阿里云 | 云效 Maven](https://developer.aliyun.com/mvn/guide) 
- [阿里云 | 云效制品仓库 Package 官方文档](https://thoughts.aliyun.com/sharespace/5e8c436d546fd9001aee824a/docs/5e8c436d546fd9001aee8244?spm=a2c4g.11186623.2.2.5db21b3frHmIbF)
- [阿里云 | 云效制品仓库 Package 地址](https://packages.aliyun.com/maven)
- [Android 开发者 | Android Maven Publish docs]( https://developer.android.com/studio/build/maven-publish-plugin)
- [Gradle 用户指南 | Gradle Maven Publish official page]( https://docs.gradle.org/current/userguide/publishing_maven.html)
- [What is Maven](http://maven.apache.org/what-is-maven.html) 
- [菜鸟教程 | Maven 教程](https://www.runoob.com/maven/maven-pom.html)
- [Publish an Android library to Maven with aar and source jar](https://stackoverflow.com/questions/26874498/publish-an-android-library-to-maven-with-aar-and-source-jar)
- [ Android 开发者 | 添加构建依赖项 ](https://developer.android.com/studio/build/dependencies)
- [Gradle 用户指南 | Gradle中的依赖管理](https://docs.gradle.org/current/userguide/dependency_management.html#dependency_management_in_gradle)



