---
title: maven 插件笔记
date: 2019-11-07 23:29:21
tags:
 - maven
 - 笔记
categories:
 - 技术笔记
---

maven 相关的插件使用

## jdk 等级 
在新建项目的时候，总是 `maven` 默认使用的编译等级和 `jdk` 为全局配置的，默认是 `5`  
可以修改全局配置，或者项目的 `pom`文件，推荐后一种，因为无法保证项目的每个人都会去配相同的全局配置，所以，一些配置尽量在项目配置里面完成  
添加以下配置  

```xml
<properties>
  <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
  <maven.compiler.source>1.8</maven.compiler.source>
  <maven.compiler.target>1.8</maven.compiler.target>
  <maven.compiler.compilerVersion>1.8</maven.compiler.compilerVersion>
</properties>
```

## 项目报告 
检查代码重复率，findbugs，p3c检查等
```xml
  <plugins>
    <plugin>
				<groupId>org.apache.maven.plugins</groupId>
				<artifactId>maven-pmd-plugin</artifactId>
				<version>3.8</version>
				<configuration>
					<rulesets>
						<ruleset>rulesets/java/ali-comment.xml</ruleset>
						<ruleset>rulesets/java/ali-concurrent.xml</ruleset>
						<ruleset>rulesets/java/ali-constant.xml</ruleset>
						<ruleset>rulesets/java/ali-exception.xml</ruleset>
						<ruleset>rulesets/java/ali-flowcontrol.xml</ruleset>
						<ruleset>rulesets/java/ali-naming.xml</ruleset>
						<ruleset>rulesets/java/ali-oop.xml</ruleset>
						<ruleset>rulesets/java/ali-orm.xml</ruleset>
						<ruleset>rulesets/java/ali-other.xml</ruleset>
						<ruleset>rulesets/java/ali-set.xml</ruleset>
					</rulesets>
					<printFailingErrors>true</printFailingErrors>
				</configuration>
				<executions>
					<execution>
						<goals>
							<goal>check</goal>
						</goals>
					</execution>
				</executions>
				<dependencies>
					<dependency>
						<groupId>com.alibaba.p3c</groupId>
						<artifactId>p3c-pmd</artifactId>
						<version>1.3.0</version>
					</dependency>
				</dependencies>
			</plugin>

			<!-- 可以使用 mvn site 生成报告，包含重复代码检测，p3c 检查，基本信息 -->
			<plugin>
				<groupId>org.apache.maven.plugins</groupId>
				<artifactId>maven-site-plugin</artifactId>
				<version>3.7.1</version>
			</plugin>
  </plugins>

<!-- 生成项目报告 -->
	<reporting>
		<plugins>
			<!-- 项目总览，包括基本信息，依赖，使用的插件等 -->
			<plugin>
				<groupId>org.apache.maven.plugins</groupId>
				<artifactId>maven-project-info-reports-plugin</artifactId>
				<version>3.0.0</version>
			</plugin>

			<!-- 源码关联，当使用检测插件的时候，发现问题代码行数，可以直接点击行数去看源码 -->
			<plugin>
				<groupId>org.apache.maven.plugins</groupId>
				<artifactId>maven-jxr-plugin</artifactId>
				<version>3.0.0</version>
			</plugin>

			<!-- p3c 的代码质量检测 -->
			<plugin>
				<groupId>org.apache.maven.plugins</groupId>
				<artifactId>maven-pmd-plugin</artifactId>
				<version>3.8</version>
				<configuration>
					<rulesets>
						<ruleset>rulesets/java/ali-comment.xml</ruleset>
						<ruleset>rulesets/java/ali-concurrent.xml</ruleset>
						<ruleset>rulesets/java/ali-constant.xml</ruleset>
						<ruleset>rulesets/java/ali-exception.xml</ruleset>
						<ruleset>rulesets/java/ali-flowcontrol.xml</ruleset>
						<ruleset>rulesets/java/ali-naming.xml</ruleset>
						<ruleset>rulesets/java/ali-oop.xml</ruleset>
						<ruleset>rulesets/java/ali-orm.xml</ruleset>
						<ruleset>rulesets/java/ali-other.xml</ruleset>
						<ruleset>rulesets/java/ali-set.xml</ruleset>
					</rulesets>
					<printFailingErrors>true</printFailingErrors>
				</configuration>
			</plugin>

			<plugin>
				<groupId>org.codehaus.mojo</groupId>
				<artifactId>findbugs-maven-plugin</artifactId>
				<version>3.0.5</version>
				<configuration>
					<!-- <configLocation>${basedir}/springside-findbugs.xml</configLocation> -->
					<!-- Low、Medium和High (Low最严格) -->
					<threshold>Medium和High</threshold>
					<!-- 设置分析工作的等级，可以为Min、Default和Max -->
					<effort>Default</effort>
					<findbugsXmlOutput>true</findbugsXmlOutput>
					<findbugsXmlOutputDirectory>target/site</findbugsXmlOutputDirectory>
				</configuration>
			</plugin>
		</plugins>
	</reporting>
```

## maven 多模块的版本号管理问题
当存在多模块，每个模块之间还有依赖的时候，如果要升级版本号，则需要很多地方都要改动。  

使用配置，和如下规则，避免  
```xml
<!-- 根目录执行 mvn versions:set -DnewVersion=1.0.0.RELEASE 修改版本号 -->
<!-- 子模块不要自己定义独立的版本号，公用父模块的版本号-->
<!-- 子模块相互依赖，一律使用 project.version 定义版本号 -->
<plugin>
    <groupId>org.codehaus.mojo</groupId>
    <artifactId>versions-maven-plugin</artifactId>
    <version>2.3</version>
    <configuration>
        <generateBackupPoms>false</generateBackupPoms>
    </configuration>
</plugin>
```

这个配置放到顶层项目。使用  
```sh
mvn versions:set -DnewVersion=1.0.0.RELEASE
```

修改版本号