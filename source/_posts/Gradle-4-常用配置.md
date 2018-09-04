---
moreLoc: 1
path: 'Android/Gradle 4 - 常用配置.md'
title: 'Gradle 4 - 常用配置'
date: 2017-12-03T15:27:03.000Z
updated: 2017-12-03T15:27:03.000Z
tags:
    - Android
categories:
    - Android
---

app/build.gradle

<!--more-->


```gradle
android {
    defaultConfig {
        // 默认配置项
    }

	buildTypes {
		// 编译配置，release或debug版本的内容
		release {
			// 为发布版本启用混淆
			minifyEnabled true  
			// 混淆文件
			proguardFiles getDefaultProguardFile('proguard-android.txt'), 'proguard-rules.pro'
			// 配置好siningConfig之后注意添加
			signingConfig signingConfigs.release
		}
		debug {
			// 将debug版本的包名设置为.debug以便能够同时在一台设备上安装debug和release版本的apk。
			applicationIdSuffix ".debug"
			// 配置好siningConfig之后注意添加
			signingConfig signingConfigs.debug
		}
	}

    compileOptions {
		// Java 的版本配置
		//配置使用JDK1.8（8），比如需要使用Lambda特性时，那么就可以在这里进行JDK版本的配置
		sourceCompatibility JavaVersion.VERSION_1_8
		targetCompatibility JavaVersion.VERSION_1_8
    }

    sourceSets {
        // 源码设置(项目目录结构的设置)
		main {
			manifest.srcFile 'AndroidManifest.xml'
			java.srcDirs = ['src']
			resources.srcDirs = ['src']
			aidl.srcDirs = ['src']
			renderscript.srcDirs = ['src']
			res.srcDirs = ['res']
			assets.srcDirs = ['assets']
			jniLibs.srcDirs = ['libs']
		}
    }

    packagingOptions {
		// 打包时的相关配置
		// 当项目中依赖的第三方库越来越多时，有可能会出现两个依赖库中存在同一个（名称）文件。如果这样，Gradle在打包时就会提示错误（警告）。
		// 那么就可以根据提示，然后使用以下方法将重复的文件剔除。
	   
		//这个是在同时使用butterknife、dagger2做的一个处理。同理，遇到类似的问题，只要根据gradle的提示，做类似处理即可。
		exclude 'META-INF/services/javax.annotation.processing.Processor'
    }

    lintOptions {
		// 编译的 lint 开关，程序在buid的时候，会执行lint检查，有任何的错误或者警告提示，都会终止构建，我们可以将其关掉。
		// abortOnError false
    }

    productFlavors {
        // 产品发布的一些东西，比如渠道、包名等
		// 这个配置是经常会使用到的，通常在适配多个渠道的时候，需要为特定的渠道做部分特殊的处理，比如设置不同的包名、应用名等。
		// 场景：当我们使用友盟统计时，通常需要设置一个渠道ID，那么我们就可以利用productFlavors来生成对应渠道信息的包
        wandoujia {
			//豌豆荚渠道包配置
			manifestPlaceholders = [UMENG_CHANNEL_VALUE: "wandoujia"]
			//manifestPlaceholders的使用在后续章节（AndroidManifest里的占位符）中介绍
        }
        xiaomi {
            manifestPlaceholders = [UMENG_CHANNEL_VALUE: "xiaomi"]
            applicationId "com.wiky.gradle.xiaomi" //配置包名

        }
        _360 {
            manifestPlaceholders = [UMENG_CHANNEL_VALUE: "_360"]
        }
        //...
    }
	
	// 如果需要在不同渠道统一配置，可以使用productFlavors.all字段
	productFlavors.all { 
        //批量修改，类似一个循序遍历
        flavor -> flavor.manifestPlaceholders = [UMENG_CHANNEL_VALUE: name] 
    }

    signingConfigs {
        //签名的配置
        release {
            //签名证书文件
            storeFile file("release.keystore")
            //别名
            keyAlias "release"
            //key的密码
            keyPassword "123456"
            //证书的密码
            storePassword "123456"
        }
        debug {
			// No debug config
        }
    }
}
```

### 批量修改生成的apk文件名

```gradle
//获取并格式化时间信息，注意，应该是与android{}同一层级
def buildTime() {
    return new Date().format("yyyy_MM_dd", TimeZone.getTimeZone("UTC"))
}

android {
	buildTypes {
		release {
			//只针对release版本进行配置
			applicationVariants.all { variant ->
				variant.outputs.each { output ->
					def outputFile = output.outputFile
					if (outputFile != null && outputFile.name.endsWith('.apk')) {
						def fileName;
                        if (variant.buildType.name.equals('release')) {
                            // 输出apk名称为Gradle_v1.0_20150115-120131_wandoujia.apk
                            fileName = "Gradle_v${defaultConfig.versionName}_${getCurrentTimeText()}.apk"
                            //过滤空格字符串为下划线
                            fileName = fileName.replaceAll(" ", "_");
                        } else if (variant.buildType.name.equals('debug')) {
                            fileName = "debug.apk"
                        }
                        output.outputFile = new File(outputFile.parent, fileName)
					}
				}
			}
		}
	}
}
```

### AndroidManifest里的占位符

AndroidManifest.xml就不在这里介绍了。我们知道，项目中的很多配置都需要在这里定义。而当在需要生成多个版本并要为每个版本配置相应的属性值时，以我们最基本的做法（每生成一个版本前修改对应的属性值）来看，这将是一件麻烦的事情。现在Gradle为我们提供了一个解决方案，就是利用manifestPlaceholders，进行动态赋值。它的使用有点类似HashMap。现在还是以使用友盟统计时的配置举例：在友盟统计分析中，我们需要在AndroidManifest里配置一个name为UMENG_CHANNEL的meta-data，这样这个meta-data的值就表示这个apk是哪个渠道，如配置小米应用商店：

`<meta-data android:value="xiaomi" android:name="UMENG_CHANNEL"/>`

显然，这个value值，不同版本的包都需要不同，所以以下为了能够使用manifestPlaceholders进行动态替换，我们需要换一种写法：

`<meta-data android:value="${UMENG_CHANNEL_VALUE}" android:name="UMENG_CHANNEL"/>`

如上${UMENG_CHANNEL_VALUE}就是一个占位符，然后我们在gradle的defaultConfig；里这样定义脚本：

```gradle
android {
    defaultConfig {
        //可以在默认配置中设置，以下的语法格式应该比较清晰，就不再介绍了。
        manifestPlaceholders = [UMENG_CHANNEL_VALUE: 'xiaomi']
    }
}
```

我们回顾一下上文中介绍productFlavors时，对每个渠道包进行配置时所举的例子正是利用manifestPlaceholders配置UMENG_CHANNEL_VALUE的值，如：

```gradle
android {  
    productFlavors {
        wandoujia {
            //豌豆荚渠道包配置
            manifestPlaceholders = [UMENG_CHANNEL_VALUE: "wandoujia"]
              //另外写法也有manifestPlaceholders.put("UMENG_CHANNEL_VALUE",'xiaomi')，稍有区别。
        }
        //...
    }  
}
```

通过以上例子，我们可以发现，AndroidManifest中的一些属性值也是可以动态改变的，这样在对特定包进行特定处理也将变得十分方便，更多的用法依次类推即可。

### 自定义你的BuildConfig

BuildConfig.Java是Android Gradle自动生成的一个java类文件，无法手动编译，但是可以通过Gradle控制，也就是说他是动态可配置的。比如在build.gradle中配置buildConfigField “boolean”, “LOG_DEBUG”, “false”后，我们点击“Sync now”后，即可在代码中使用BuildConfig.java新加的LOG_DEBUG属性，如：

```
public final class BuildConfig {
  //...
  // Fields from build type: debug
  public static final boolean LOG_DEBUG = true;
}
```

buildConfigField 一共有3个参数，第一个是数据类型，即新增成员变量的类型（与Java的类型是对等）；第二个参数是成员变量名，如：LOG_DEBUG；第三个参数是常量值。

以下我们介绍一种常见的用法，场景：我们需要在debug版本包中输出log，而不希望在release版本中输出log，那么就可以通过以下方法进行配置（当然，我们代码中的log开关是使用BuildConfig中相关属性进行设置的）：

```
buildTypes {
     debug {
        // 显示Log
        buildConfigField "boolean", "LOG_DEBUG", "true"
        //...
     }

     release {
        // 不显示Log
        buildConfigField "boolean", "LOG_DEBUG", "false"
        //...
     }
}
```

另外，还用一种常见的情况是：当我们的服务器有debug和release环境时，我们可能也需要在App对应的版本中配置对应的信息。如在debug版本中配置服务端debug环境对应的url。这样我们在开发的过程中将减少很多麻烦的工作。具体做法就不再细说了。

### 动态配置 Android resValue

在你的 gradle 内容 buildTypes 或者 productFlavors 下面，如 release 体内写上类似：
resValue “string”, “AppName”, “Tel4Drakeet”
意为把名为 AppName 的 string value 值改为 Tel4Drakeet

