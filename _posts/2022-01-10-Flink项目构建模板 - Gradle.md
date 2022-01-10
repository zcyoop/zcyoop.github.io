---
layout: post
title:  'Flink项目构建模板 - Gradle'
date:   2022-01-10
tags: [flink]
categories: [大数据框架,flink]
---

# Flink项目构建模板 - Gradle

> 环境
>
> - Gradle:7.x
> - Java:8
>
> 相较于[官方模板](https://nightlies.apache.org/flink/flink-docs-master/docs/dev/datastream/project-configuration/#gradle)
>
> - 升级[Gradle Shadow](https://github.com/johnrengelman/shadow#gradle-shadow)版本
> - 移出过时仓库`JCenter`
> - 修改过时语法
> - 仓库改为阿里镜像

```groovy
buildscript {
    repositories {
        maven {
            // 源仓库地址：https://plugins.gradle.org/m2/
            url "https://maven.aliyun.com/repository/gradle-plugin"
        }
    }
    dependencies {
        classpath "gradle.plugin.com.github.johnrengelman:shadow:7.1.2"
    }
}


plugins {
    id 'java'
    id 'application'
    // shadow plugin to produce fat JARs
    id 'com.github.johnrengelman.shadow' version '7.1.2'
}

//group 'org.example'
version '1.0-SNAPSHOT'
mainClassName = 'org.myorg.quickstart.StreamingJob'

ext {
    javaVersion = '1.8'
    flinkVersion = '1.14.2'
    scalaBinaryVersion = '2.11'
    slf4jVersion = '1.7.32'
    log4jVersion = '2.17.1'
}

sourceCompatibility = javaVersion
targetCompatibility = javaVersion


repositories {
    maven {
        url 'https://maven.aliyun.com/repository/public'
    }
    mavenCentral()
}

configurations {
    flinkShadowJar // dependencies which go into the shadowJar

    // always exclude these (also from transitive dependencies) since they are provided by Flink
    flinkShadowJar.exclude group: 'org.apache.flink', module: 'force-shading'
    flinkShadowJar.exclude group: 'com.google.code.findbugs', module: 'jsr305'
    flinkShadowJar.exclude group: 'org.slf4j'
    flinkShadowJar.exclude group: 'org.apache.logging.log4j'
}

dependencies {
    // --------------------------------------------------------------
    // Compile-time dependencies that should NOT be part of the
    // shadow jar and are provided in the lib folder of Flink
    // --------------------------------------------------------------
    implementation "org.apache.flink:flink-streaming-java_${scalaBinaryVersion}:${flinkVersion}"

    // --------------------------------------------------------------
    // Dependencies that should be part of the shadow jar, e.g.
    // connectors. These must be in the flinkShadowJar configuration!
    // --------------------------------------------------------------
    //flinkShadowJar "org.apache.flink:flink-connector-kafka:${flinkVersion}"

    implementation "org.apache.logging.log4j:log4j-api:${log4jVersion}"
    implementation "org.apache.logging.log4j:log4j-core:${log4jVersion}"
    implementation "org.apache.logging.log4j:log4j-slf4j-impl:${log4jVersion}"
    implementation "org.slf4j:slf4j-log4j12:${slf4jVersion}"

}

test {
    useJUnitPlatform()
}

shadowJar {
    configurations = [project.configurations.flinkShadowJar]
}
```

