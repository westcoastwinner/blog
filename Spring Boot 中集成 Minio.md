# Spring Boot 中集成 Minio

在 Spring Boot 中集成 Minio 实现文件存储是一个非常经典且实用的方案，特别是在微服务架构中。Minio 兼容 Amazon S3 协议，因此我们可以使用官方的 Minio Java SDK 来轻松操作。

以下是实现文件上传和下载的完整步骤指南。

------

### 1. 核心架构与流程

在开始写代码之前，理解数据流向很有帮助。Spring Boot 作为服务端接收前端请求，通过 Minio Client 将文件流传输到 Minio Server。

### 2. 引入依赖 (Maven)

首先，在你的 `pom.xml` 文件中添加 Minio 的官方 SDK 依赖。

XML

```
<dependencies>
    <dependency>
        <groupId>io.minio</groupId>
        <artifactId>minio</artifactId>
        <version>8.5.7</version> </dependency>
    
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
</dependencies>
```

### 3. 配置文件 (application.yml)

在 `application.yml` 中配置连接 Minio 所需的参数。

YAML

```
spring:
  servlet:
    multipart:
      max-file-size: 10MB      # 单个文件大小限制
      max-request-size: 10MB   # 总请求大小限制

minio:
  endpoint: http://localhost:9000  # Minio 服务地址
  accessKey: minioadmin            # 你的 Access Key
  secretKey: minioadmin            # 你的 Secret Key
  bucketName: my-data-bucket       # 存储桶名称
```

### 4. 创建配置类 (Config)

我们需要创建一个 `@Configuration` 类，将 `MinioClient` 注入到 Spring 容器中，以便在其他地方复用。

Java

```
import io.minio.MinioClient;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class MinioConfig {

    @Value("${minio.endpoint}")
    private String endpoint;

    @Value("${minio.accessKey}")
    private String accessKey;

    @Value("${minio.secretKey}")
    private String secretKey;

    @Bean
    public MinioClient minioClient() {
        return MinioClient.builder()
                .endpoint(endpoint)
                .credentials(accessKey, secretKey)
                .build();
    }
}
```

------

### 5. 封装工具类 (Service/Util)

为了代码的整洁，建议将 Minio 的具体操作（上传、下载、检查 Bucket）封装在一个 Service 或 Util 类中。

Java

```
import io.minio.*;
import io.minio.http.Method;
import jakarta.annotation.PostConstruct; // Spring Boot 3.x 使用 jakarta, 2.x 使用 javax
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Service;
import org.springframework.web.multipart.MultipartFile;

import java.io.InputStream;

@Service
public class MinioService {

    @Autowired
    private MinioClient minioClient;

    @Value("${minio.bucketName}")
    private String bucketName;

    /**
     * 初始化时检查 Bucket 是否存在，不存在则创建
     */
    @PostConstruct
    public void init() {
        try {
            boolean isExist = minioClient.bucketExists(BucketExistsArgs.builder().bucket(bucketName).build());
            if (!isExist) {
                minioClient.makeBucket(MakeBucketArgs.builder().bucket(bucketName).build());
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    /**
     * 上传文件
     */
    public String uploadFile(MultipartFile file) throws Exception {
        // 建议使用 UUID 重命名文件防止覆盖，这里为了演示使用原文件名
        String fileName = file.getOriginalFilename();
        
        InputStream inputStream = file.getInputStream();
        
        minioClient.putObject(
                PutObjectArgs.builder()
                        .bucket(bucketName)
                        .object(fileName)
                        .stream(inputStream, file.getSize(), -1)
                        .contentType(file.getContentType())
                        .build()
        );
        return fileName;
    }

    /**
     * 获取文件流（用于下载）
     */
    public InputStream downloadFile(String fileName) throws Exception {
        return minioClient.getObject(
                GetObjectArgs.builder()
                        .bucket(bucketName)
                        .object(fileName)
                        .build()
        );
    }
    
    /**
     * 获取文件预览/下载链接 (带过期时间)
     */
    public String getPresignedUrl(String fileName) throws Exception {
        return minioClient.getPresignedObjectUrl(
                GetPresignedObjectUrlArgs.builder()
                        .bucket(bucketName)
                        .object(fileName)
                        .method(Method.GET)
                        .expiry(24 * 60 * 60) // 24小时有效
                        .build()
        );
    }
}
```

------

### 6. 控制层 (Controller)

最后，创建 REST 接口供前端调用。

Java

```
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.core.io.InputStreamResource;
import org.springframework.http.HttpHeaders;
import org.springframework.http.MediaType;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;
import org.springframework.web.multipart.MultipartFile;

import java.io.InputStream;
import java.net.URLEncoder;
import java.nio.charset.StandardCharsets;

@RestController
@RequestMapping("/files")
public class FileController {

    @Autowired
    private MinioService minioService;

    // 上传接口
    @PostMapping("/upload")
    public ResponseEntity<String> upload(@RequestParam("file") MultipartFile file) {
        try {
            String fileName = minioService.uploadFile(file);
            return ResponseEntity.ok("文件上传成功: " + fileName);
        } catch (Exception e) {
            return ResponseEntity.status(500).body("上传失败: " + e.getMessage());
        }
    }

    // 下载接口
    @GetMapping("/download/{fileName}")
    public ResponseEntity<InputStreamResource> download(@PathVariable String fileName) {
        try {
            InputStream inputStream = minioService.downloadFile(fileName);
            
            // 构建响应头，强制浏览器下载而不是直接打开
            return ResponseEntity.ok()
                    .header(HttpHeaders.CONTENT_DISPOSITION, "attachment;filename=" + 
                            URLEncoder.encode(fileName, StandardCharsets.UTF_8))
                    .contentType(MediaType.APPLICATION_OCTET_STREAM)
                    .body(new InputStreamResource(inputStream));
        } catch (Exception e) {
            return ResponseEntity.status(404).build();
        }
    }
}
```

------

### 关键注意事项

1. **异常处理**: 实际生产代码中，应该捕获具体的 Minio 异常（如 `ErrorResponseException`）并返回统一的错误响应结构。
2. **文件名策略**: 不要直接使用用户上传的文件名。为了防止文件名冲突，通常使用 `UUID` + `原始扩展名` 的方式重命名文件，并将原始文件名和存储路径的映射关系保存在数据库中。
3. **大文件上传**: 对于超大文件（如几百 MB 或 GB 级别），Minio 支持分片上传（Multipart Upload），上述代码是基础的流式上传，适合中小文件。
4. **预览 vs 下载**:
   - 如果你希望图片直接在浏览器显示，`Content-Disposition` 应该设为 `inline`。
   - 如果你希望强制下载，`Content-Disposition` 应该设为 `attachment`（如上面代码所示）。

**您想在这个基础上了解如何实现“分片断点续传”或者如何将文件元数据（上传者、时间等）保存到数据库吗？**