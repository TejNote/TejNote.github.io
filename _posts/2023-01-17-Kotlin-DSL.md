---
title:  "Kotlin DSL 변경"
excerpt: "Kotlin DSL 변경에 관련된 사항을 적어보았습니다."

categories:
  - Android
tags:
  - Android
  - MultiModule
  - Module
  - Version
  - Category
last_modified_at: 2023-01-17T21:00:00+09:00
---

## Kotlin DSL이란?

DSL은 Domain-Specific Language로써 특정 영역에 대한 연산 및 작업을 간결하게 기술할 수 있는 언어 입니다.
흔히 말하는 Query인 SQL 또한 DSL이라고 할수 있습니다.
그중 Kotlin DSL은 코틀린의 언어적인 특징으로 가독성이 좋고 간략한 코드를 사용하여 Gradle을 작성할 수 있도록 하는 것을 이야기 합니다.

## Kotlin DSL을 쓰는 이유?

1. 컴파일 시 에러 확인 가능
2. 자동 완성으로 코드 작성 가능
3. IDE 지원으로 편리한 수정 가능

마지막으로 최종인 Version Category를 사용하기 위해서 예제를 확인하던 중 대부분이 Kotlin DSL을 기준으로 되어 있어서 변경이 필수적으로 필요하게 되었습니다. 

## Groovy DSL에서 Kotlin DSL로 변경

1. 프로젝트 단위 build.gradle 수정
    1. 파일명을 build.gradle에서 build.gradle.kts로 변경
    2. 작은 따옴표를 큰따옴표로 변경
    3. `buildscript` 변경
        
        ```kotlin
        // 이전
        buildscript {
            repositories {
        			mavenCentral()
        			google()
        			mavenCentral()
        			gradlePluginPortal()
            }
        
            dependencies {
                classpath 'com.android.tools.build:gradle:7.2.2'
                classpath 'com.google.gms:google-services:4.3.13'
                classpath 'org.jetbrains.kotlin:kotlin-gradle-plugin:1.6.21'
            }
        }
        
        // 변경
        buildscript {
        	repositories {
        		google()
        		mavenCentral()
        		gradlePluginPortal()
        	}
        
        	dependencies {
        			classpath("com.android.tools.build:gradle:7.2.2")
        	    classpath("com.google.gms:google-services:4.3.13")
        	    classpath("org.jetbrains.kotlin:kotlin-gradle-plugin:1.6.21")
        	}
        }
        ```
        
    4. `allprojects` 변경
        
        ```kotlin
        // 기존
        allprojects {
            repositories {
        				mavenCentral()
                google()
        				gradlePluginPortal()
                maven { url 'https://maven.google.com' }
            }
        }
        
        // 변경
        allprojects {
        	repositories {
        			mavenCentral()
              google()
        			gradlePluginPortal()
              maven { url 'https://maven.google.com' }
        	}
        }
        ```
        
    5. `clean` 변경
        
        ```kotlin
        // 기존
        task clean(type: Delete) {
            delete rootProject.buildDir
        }
        
        // 변경
        tasks.register("clean", Delete::class) {
        	delete(rootProject.buildDir)
        }
        ```
        

1. 모듈 단위 build.gradle 수정
    1. 파일명을 build.gradle에서 build.gradle.kts로 변경
    2. 작은 따옴표를 큰 따옴표로 변경
    3. `plugins` 변경
        
        ```kotlin
        // 기존 
        apply plugin: 'com.android.application'
        apply plugin: 'com.google.gms.google-services'
        apply plugin: 'kotlin-android'
        
        // 변경
        plugins {
        	id("com.android.application")
        	id("com.google.gms.google-services")
        	id("kotlin-android")
        }
        ```
        
    4. *`android`* 설정 변경
        
        ```kotlin
        // 기존
        compileSdkVersion 31
        flavorDimensions "default" // flavor를 쓸 경우에만 사용
        
        // 변경
        compileSdk = 31
        flavorDimensions.add("flavors") // flavor를 쓸 경우에만 사용
        ```
        
    5. `defaultConfig` 변경
        
        ```kotlin
        // 기존
        applicationId 'com.tejnote.test'
        minSdkVersion 21
        targetSdkVersion 31
        versionCode 1
        versionName "1.0.0"
        vectorDrawables.useSupportLibrary = true
        
        // 변경
        applicationId = "com.tejnote.test"
        minSdk = 21
        targetSdk = 31
        versionCode = 1
        versionName = "1.0.0"
        vectorDrawables.useSupportLibrary = true
        ```
        
    6. `compileOptions` 변경
        
        ```kotlin
        // 기존
        compileOptions {
        	sourceCompatibility JavaVersion.VERSION_1_8
        	targetCompatibility JavaVersion.VERSION_1_8
        }
        
        // 변경 : Java 11로 올림
        compileOptions {
        	sourceCompatibility = JavaVersion.VERSION_11
        	targetCompatibility = JavaVersion.VERSION_11
        }
        
        kotlinOptions {
        	jvmTarget = "11"
        }
        ```
        
    7.   `productFlavors` 변경
        
        ```kotlin
        // 기존
        productFlavors {
        		dev {
        		applicationIdSuffix ".dev"
        		aaptOptions.cruncherEnabled = false
        
        		// Manifest 변수화 시 사용
        		manifestPlaceholders = [appName   : "@string/app_name_dev"]
        		// Build 변수
        		buildConfigField('boolean', 'IS_RELEASE', "false")
        	}
        
        	prod {
        		// Manifest 변수화 시 사용
        		manifestPlaceholders = [appName   : "@string/app_name_prod"]
        		// Build 변수
        		buildConfigField('boolean', 'IS_RELEASE', "true")
        	}
        }
        
        sourceSets {
        	dev {
        		java.srcDirs = ['src/main/java', 'src/dev/java']
        	}
        	prod {
        		java.srcDirs = ['src/main/java', 'src/prod/java']
        	}
        }
        // 변경
        productFlavors {
        	create("dev") {
        		dimension = "flavors"
        		applicationIdSuffix = ".dev"
        		aaptOptions.cruncherEnabled = false
        
        		// Manifest 변수화 시 사용
        		manifestPlaceholders["appName"] = "@string/app_name_dev"
        		// Build 변수
        		buildConfigField("boolean", "IS_RELEASE", false.toString())
        	}
        
        	create("prod") {
        		dimension = "flavors"
        
        		// Manifest 변수화 시 사용
        		manifestPlaceholders["appName"] = "@string/app_name_prod"
        		// Build 변수
        		buildConfigField("boolean", "IS_RELEASE", true.toString())
        	}
        }
        
        // Flavor에 따른 코드 분기 처리
        sourceSets {
        	getByName("dev") {
        		java.srcDirs("src/dev/java")
        	}
        	getByName("prod") {
        		java.srcDirs("src/prod/java")
        	}
        }
        ```
        
    8.   `buildTypes` 변경
        
        ```kotlin
        // 기존
        buildTypes {
        	debug {
        		signingConfig signingConfigs.debug
        		zipAlignEnabled false
        		minifyEnabled false
        		shrinkResources false
        		debuggable true
        
        		proguardFiles getDefaultProguardFile('proguard-android.txt'), 'proguard-rules.pro'
        	}
        	release {
        		signingConfig signingConfigs.release
        		zipAlignEnabled true
        		minifyEnabled true
        		shrinkResources true
        		debuggable false
        
        		proguardFiles getDefaultProguardFile('proguard-android.txt'), 'proguard-rules.pro'
        	}
        }
        
        // 변경
        buildTypes {
        	getByName("debug") {
        		signingConfig = signingConfigs.getByName("debug")
        		isMinifyEnabled = false
        		isShrinkResources = false
        		isDebuggable = true
        
        		proguardFile(getDefaultProguardFile("proguard-android.txt"))
        		proguardFile(file("proguard-rules.pro"))
        	}
        	getByName("release") {
        		signingConfig = signingConfigs.getByName("release")
        		isMinifyEnabled = true
        		isShrinkResources = true
        		isDebuggable = false
        
        		proguardFile(getDefaultProguardFile("proguard-android.txt"))
        		proguardFile(file("proguard-rules.pro"))
        	}
        }
        ```
        
    9. `dependencies` 변경
        
        ```kotlin
        // 기존 
        implementation 'androidx.constraintlayout:constraintlayout:2.1.4'
        implementation 'androidx.viewpager2:viewpager2:1.0.0'
        
        // 변경
        implementation("androidx.constraintlayout:constraintlayout:2.1.4")
        implementation("androidx.viewpager2:viewpager2:1.0.0")
        ```
        
2. 공통 사항
    1. ‘ 작은 따옴표는 “ 큰 따옴표로 변경
    2. “변수 값” 표기는 “변수 = 값” 표기로 변경
    3. boolean일 경우 is 용어 추가

## 결과

Kotlin DSL로 옮기면서 처음에는 굳이 왜 바꿔야 하는지 의문이 들었고, Version Category를 하기위해 어쩔수 없지라고 생각했습니다.
하지만 Version Category는 Groovy DSL에서도 가능하다는 걸 뒤늦게 알게 되었지만 변경한 것을 후회하지 않습니다.  
이유는 Groovy 의 경우 제가 사용해보지 않은 언어라서 인터넷에서 공부한 내용만 가지고 적용했을 뿐 어떻게 작성해야하는지 고민하지 않았지만, 
Kotlin DSL로 변경하면서는 이해하는 언어로 변경되어 왜 이렇게 작성하는지에 대한 이해도가 더 증가 되었습니다.
또한 어느정도 자동완성을 지원해줌으로써 기능을 찾기 좀 더 수월해졌습니다.