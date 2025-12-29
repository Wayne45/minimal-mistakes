---
title: "開發 Maven Plugin 的實務心得"
excerpt: "Maven Plugin 不只是寫程式，更牽涉到使用者體驗與專案整合。本文分享 Plugin 設計、參數定義與常見錯誤。"
categories:
  - DevOps
  - Java
tags:
  - Maven
  - Plugin Development
  - Build Tools
toc: true
---

## 前言

Maven Plugin 不只是寫程式，更牽涉到使用者體驗與專案整合。本文分享 Plugin 設計、參數定義與常見錯誤。

## 基本結構

### POM 配置

```xml
<project>
    <groupId>com.example</groupId>
    <artifactId>my-maven-plugin</artifactId>
    <version>1.0.0</version>
    <packaging>maven-plugin</packaging>

    <dependencies>
        <dependency>
            <groupId>org.apache.maven</groupId>
            <artifactId>maven-plugin-api</artifactId>
            <version>3.9.0</version>
        </dependency>
        <dependency>
            <groupId>org.apache.maven.plugin-tools</groupId>
            <artifactId>maven-plugin-annotations</artifactId>
            <version>3.9.0</version>
            <scope>provided</scope>
        </dependency>
    </dependencies>
</project>
```

### Mojo 實作

```java
@Mojo(name = "generate", defaultPhase = LifecyclePhase.GENERATE_SOURCES)
public class GenerateMojo extends AbstractMojo {

    @Parameter(property = "inputDir", required = true)
    private File inputDirectory;

    @Parameter(property = "outputDir", defaultValue = "${project.build.directory}/generated")
    private File outputDirectory;

    @Parameter(property = "skip", defaultValue = "false")
    private boolean skip;

    @Component
    private MavenProject project;

    @Override
    public void execute() throws MojoExecutionException {
        if (skip) {
            getLog().info("Skipping generation");
            return;
        }

        if (!inputDirectory.exists()) {
            throw new MojoExecutionException("Input directory does not exist: " + inputDirectory);
        }

        // 實際邏輯
        generate();
    }
}
```

## 設計要點

### 1. 參數設計

```java
// 必填參數
@Parameter(required = true)
private String apiKey;

// 預設值
@Parameter(defaultValue = "UTF-8")
private String encoding;

// 從 property 讀取（mvn -Dmy.property=value）
@Parameter(property = "my.property")
private String myProperty;

// 複雜類型
@Parameter
private List<String> includes;

@Parameter
private Map<String, String> properties;
```

### 2. 錯誤處理

```java
@Override
public void execute() throws MojoExecutionException, MojoFailureException {
    try {
        doExecute();
    } catch (IOException e) {
        // MojoExecutionException: 技術錯誤，建置失敗
        throw new MojoExecutionException("Failed to process files", e);
    } catch (ValidationException e) {
        // MojoFailureException: 邏輯錯誤，可能是使用者設定問題
        throw new MojoFailureException("Validation failed: " + e.getMessage());
    }
}
```

### 3. 日誌輸出

```java
getLog().debug("Processing file: " + file);  // -X 才會顯示
getLog().info("Generated 10 files");          // 正常輸出
getLog().warn("Deprecated feature used");     // 警告
getLog().error("Failed to generate");         // 錯誤
```

## 常見錯誤

1. **忘記 @Mojo 註解**：Plugin 無法被識別
2. **參數類型不匹配**：File vs String
3. **沒有處理 null**：Optional 參數可能是 null
4. **Phase 選擇錯誤**：影響執行順序

## 測試方式

```java
@Test
public void testGenerate() throws Exception {
    GenerateMojo mojo = new GenerateMojo();
    setVariableValueToObject(mojo, "inputDirectory", new File("src/test/resources"));
    setVariableValueToObject(mojo, "outputDirectory", tempFolder.getRoot());

    mojo.execute();

    assertTrue(new File(tempFolder.getRoot(), "output.txt").exists());
}
```

## 結論

開發 Maven Plugin 需要考慮使用者體驗，清晰的參數設計和錯誤訊息能大幅降低使用門檻。
