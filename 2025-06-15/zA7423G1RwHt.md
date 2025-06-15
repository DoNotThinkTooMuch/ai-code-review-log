通过对提供的 Git diff 记录进行分析，可以归纳出以下几点代码审查意见：

### 1. **代码组织与职责分离**
   - 目前将 ChatCompletionRequestDTO、OpenAiChatCompletionResponseDTO 和 DeepseekChatCompletionResponseDTO 收集在同一目录下可能较为混乱，建议按模块功能分组，以便于维护和扩展。
   - `AiCodeReview` 类的责任除了处理代码审核的逻辑外，还涉及 API 调用。在未来的扩展中，可能会导致代码膨胀。建议将 API 调用逻辑提取到单独的服务类中。

### 2. **代码精简与重用**
   - `OpenAIChatCompletionRequest` 和 `DeepSeekChatCompletionRequest` 被删除并被 `ChatCompletionRequestDTO` 替代，虽然这减少了类的数量，但导致不同 API 请求的配置无法单独配置参数（如 `max_tokens`, `temperature` 等），建议考虑保留这些特定请求类以提高灵活性。

### 3. **异常处理**
   - 在 AIApiClientUtil 中，虽然捕获了异常并抛出了 RuntimeException，但没有详细记录异常堆栈。保留原始异常消息，并可以使用日志记录器来记录 API 调用的详细信息，将有助于后续的调试和问题排查。
   - 在 ApiTest 中，`e.printStackTrace()` 可以被替换为更标准的日志记录方式。

### 4. **DTO 使用的一致性**
   - `ChatCompletionRequestDTO` 中 `model` 属性被初始化为 `Model.GPT_4O_MINI.getCode()`，但未能在调用时覆盖，这可能导致在调用 API 时模型设置不准确。建议将这些属性设置为可选字段，以便于在调用时自定义。

### 5. **注释与文档**
   - 在说明中应明确指明每个方法的责任，尤其是在公共 API 调用中。为类及其方法提供 JavaDoc 风格的注释将有助于后续开发和维护。
   - 例如，AIApiClientUtil 中的 `callOpenAI` 和 `callDeepSeekOllama` 方法应该强调如何使用和配置。

### 6. **测试用例的健全性**
   - 在 `ApiTest` 中应增加更多的测试用例，以涵盖不同的输入情况（比如空请求、无效参数等），从而提高整体代码的健壮性。
   - 建议使用 Mockito 或类似的库来模拟依赖的外部服务，从而实现孤立测试。

### 7. **遵循编码规范**
   - 保持一致的命名规范，确保每个类、方法和变量的命名都表达其功能且遵循 Java 的命名惯例。

### 总结
整体而言，代码经过了一定的简化和重构，增强了对 DTO 的使用，考虑到未来的扩展性，建议进一步优化类的组织结构，同时重视异常处理与测试用例的覆盖，确保软件在各种条件下都能稳定运行。这些改进将提高代码的可维护性和可读性，为未来的团队合作打下基础。

---

## Code Changes:
```diff
diff --git a/ai-code-review-sdk/src/main/java/cn/thinktoomuch/middleware/sdk/AiCodeReview.java b/ai-code-review-sdk/src/main/java/cn/thinktoomuch/middleware/sdk/AiCodeReview.java
index 856816c..edb3e97 100644
--- a/ai-code-review-sdk/src/main/java/cn/thinktoomuch/middleware/sdk/AiCodeReview.java
+++ b/ai-code-review-sdk/src/main/java/cn/thinktoomuch/middleware/sdk/AiCodeReview.java
@@ -1,6 +1,9 @@
 package cn.thinktoomuch.middleware.sdk;
 
 import cn.thinktoomuch.middleware.sdk.domain.model.*;
+import cn.thinktoomuch.middleware.sdk.infrastructure.ai.dto.ChatCompletionRequestDTO;
+import cn.thinktoomuch.middleware.sdk.infrastructure.ai.dto.DeepseekChatCompletionResponseDTO;
+import cn.thinktoomuch.middleware.sdk.infrastructure.ai.dto.OpenAiChatCompletionResponseDTO;
 import cn.thinktoomuch.middleware.sdk.types.util.AIApiClientUtil;
 import cn.thinktoomuch.middleware.sdk.types.util.WeiXinAccessTokenUtil;
 import com.alibaba.fastjson2.JSON;
@@ -10,7 +13,6 @@ import org.eclipse.jgit.transport.UsernamePasswordCredentialsProvider;
 
 import java.io.*;
 import java.net.HttpURLConnection;
-import java.net.MalformedURLException;
 import java.net.URL;
 import java.nio.charset.StandardCharsets;
 import java.text.SimpleDateFormat;
@@ -86,41 +88,44 @@ public class AiCodeReview {
         String systemMessage = "你是一个高级编程架构师，精通各类场景方案、架构设计和编程语言请，请您根据git diff记录，对代码做出评审。代码如下:";
         // 2.openai 代码评审
 
-        OpenAIChatCompletionRequest openAiRequest = new OpenAIChatCompletionRequest();
+        ChatCompletionRequestDTO openAiRequest = new ChatCompletionRequestDTO();
         openAiRequest.setModel(Model.GPT_4O_MINI.getCode());
-        openAiRequest.setMessages(new ArrayList<ChatCompletionRequest.Prompt>() {
+        openAiRequest.setMessages(new ArrayList<ChatCompletionRequestDTO.Prompt>() {
             private static final long serialVersionUID = -7988151926241837899L;
             {
-                add(new ChatCompletionRequest.Prompt("user", systemMessage));
-                add(new ChatCompletionRequest.Prompt("user", diffCode.toString()));
+                add(new ChatCompletionRequestDTO.Prompt("user", systemMessage));
+                add(new ChatCompletionRequestDTO.Prompt("user", diffCode.toString()));
             }
         });
-
-        String openAiReviewResult = AIApiClientUtil.callOpenAI(apiKey, openAiRequest);
+        openAiRequest.setStream(false);
+        OpenAiChatCompletionResponseDTO openAiReviewResult = AIApiClientUtil.callOpenAI(apiKey, openAiRequest);
+        OpenAiChatCompletionResponseDTO.Message message = openAiReviewResult.getChoices().get(0).getMessage();
 
         System.out.println("======= OpenAI Code Review Result =======" );
         System.out.println();
-        System.out.println(openAiReviewResult);
+        System.out.println(message);
         System.out.println();
         System.out.println("======= OpenAI Code Review Result =======" );
 
-//        DeepSeekChatCompletionRequest deepSeekRequest = new DeepSeekChatCompletionRequest();
+//        ChatCompletionRequestDTO deepSeekRequest = new ChatCompletionRequestDTO();
 //        deepSeekRequest.setModel(Model.DEEPSEEK_R1_1_5B.getCode());
-//        deepSeekRequest.setMessages(new ArrayList<ChatCompletionRequest.Prompt>() {
+//        deepSeekRequest.setMessages(new ArrayList<ChatCompletionRequestDTO.Prompt>() {
 //            private static final long serialVersionUID = -7988151926241837899L;
 //            {
-//                add(new ChatCompletionRequest.Prompt("system", systemMessage));
-//                add(new ChatCompletionRequest.Prompt("user", diffCode.toString()));
+//                add(new ChatCompletionRequestDTO.Prompt("system", systemMessage));
+//                add(new ChatCompletionRequestDTO.Prompt("user", diffCode.toString()));
 //            }
 //        });
-//        String deepSeekReviewResult = AIApiClientUtil.callDeepSeekOllama(ollamaUrl, deepSeekRequest);
+//        deepSeekRequest.setStream(false);
+//        DeepseekChatCompletionResponseDTO deepSeekReviewResult = AIApiClientUtil.callDeepSeekOllama(ollamaUrl, deepSeekRequest);
+//        String deepSeekReviewContent = deepSeekReviewResult.getMessage().get(0).getContent();
 //        System.out.println("======= DeepSeek Code Review Result =======");
 //        System.out.println();
-//        System.out.println(deepSeekReviewResult);
+//        System.out.println(deepSeekReviewContent);
 //        System.out.println();
 //        System.out.println("======= DeepSeek Code Review Result =======");
 
-        return openAiReviewResult;
+        return message.getContent();
     }
 
     private static String writeLog(String token, String log)throws Exception {
diff --git a/ai-code-review-sdk/src/main/java/cn/thinktoomuch/middleware/sdk/domain/model/DeepSeekChatCompletionRequest.java b/ai-code-review-sdk/src/main/java/cn/thinktoomuch/middleware/sdk/domain/model/DeepSeekChatCompletionRequest.java
deleted file mode 100644
index 8389d47..0000000
--- a/ai-code-review-sdk/src/main/java/cn/thinktoomuch/middleware/sdk/domain/model/DeepSeekChatCompletionRequest.java
+++ /dev/null
@@ -1,28 +0,0 @@
-package cn.thinktoomuch.middleware.sdk.domain.model;
-
-import java.util.List;
-import java.util.Map;
-
-public class DeepSeekChatCompletionRequest extends ChatCompletionRequest {
-
-    private boolean stream = false;
-    private Map<String, Object> options;
-
-
-
-    public boolean isStream() {
-        return stream;
-    }
-
-    public Map<String, Object> getOptions() {
-        return options;
-    }
-
-    public void setStream(boolean stream) {
-        this.stream = stream;
-    }
-
-    public void setOptions(Map<String, Object> options) {
-        this.options = options;
-    }
-}
diff --git a/ai-code-review-sdk/src/main/java/cn/thinktoomuch/middleware/sdk/domain/model/OpenAIChatCompletionRequest.java b/ai-code-review-sdk/src/main/java/cn/thinktoomuch/middleware/sdk/domain/model/OpenAIChatCompletionRequest.java
deleted file mode 100644
index 2e43990..0000000
--- a/ai-code-review-sdk/src/main/java/cn/thinktoomuch/middleware/sdk/domain/model/OpenAIChatCompletionRequest.java
+++ /dev/null
@@ -1,33 +0,0 @@
-package cn.thinktoomuch.middleware.sdk.domain.model;
-
-import java.util.List;
-
-public class OpenAIChatCompletionRequest extends ChatCompletionRequest {
-    private int max_tokens = 2000;
-    private double temperature = 0.3;
-    private boolean stream = false;
-
-    public int getMax_tokens() {
-        return max_tokens;
-    }
-
-    public double getTemperature() {
-        return temperature;
-    }
-
-    public boolean isStream() {
-        return stream;
-    }
-
-    public void setMax_tokens(int max_tokens) {
-        this.max_tokens = max_tokens;
-    }
-
-    public void setTemperature(double temperature) {
-        this.temperature = temperature;
-    }
-
-    public void setStream(boolean stream) {
-        this.stream = stream;
-    }
-}
diff --git a/ai-code-review-sdk/src/main/java/cn/thinktoomuch/middleware/sdk/domain/model/ChatCompletionRequest.java b/ai-code-review-sdk/src/main/java/cn/thinktoomuch/middleware/sdk/infrastructure/ai/dto/ChatCompletionRequestDTO.java
similarity index 57%
rename from ai-code-review-sdk/src/main/java/cn/thinktoomuch/middleware/sdk/domain/model/ChatCompletionRequest.java
rename to ai-code-review-sdk/src/main/java/cn/thinktoomuch/middleware/sdk/infrastructure/ai/dto/ChatCompletionRequestDTO.java
index cebfd05..dd4d342 100644
--- a/ai-code-review-sdk/src/main/java/cn/thinktoomuch/middleware/sdk/domain/model/ChatCompletionRequest.java
+++ b/ai-code-review-sdk/src/main/java/cn/thinktoomuch/middleware/sdk/infrastructure/ai/dto/ChatCompletionRequestDTO.java
@@ -1,11 +1,15 @@
-package cn.thinktoomuch.middleware.sdk.domain.model;
+package cn.thinktoomuch.middleware.sdk.infrastructure.ai.dto;
+
+import cn.thinktoomuch.middleware.sdk.domain.model.Model;
 
 import java.util.List;
 
-public class ChatCompletionRequest {
+public class ChatCompletionRequestDTO {
     private String model = Model.GPT_4O_MINI.getCode();
     private List<Prompt> messages;
-
+    private Double temperature;
+    private Integer max_tokens;
+    private Boolean stream;
 
     public static class Prompt {
         private String role;
@@ -52,4 +56,28 @@ public class ChatCompletionRequest {
     public void setMessages(List<Prompt> messages) {
         this.messages = messages;
     }
+
+    public Double getTemperature() {
+        return temperature;
+    }
+
+    public Integer getMax_tokens() {
+        return max_tokens;
+    }
+
+    public Boolean getStream() {
+        return stream;
+    }
+
+    public void setTemperature(Double temperature) {
+        this.temperature = temperature;
+    }
+
+    public void setMax_tokens(Integer max_tokens) {
+        this.max_tokens = max_tokens;
+    }
+
+    public void setStream(Boolean stream) {
+        this.stream = stream;
+    }
 }
diff --git a/ai-code-review-sdk/src/main/java/cn/thinktoomuch/middleware/sdk/infrastructure/ai/dto/DeepseekChatCompletionResponseDTO.java b/ai-code-review-sdk/src/main/java/cn/thinktoomuch/middleware/sdk/infrastructure/ai/dto/DeepseekChatCompletionResponseDTO.java
new file mode 100644
index 0000000..9ae12ef
--- /dev/null
+++ b/ai-code-review-sdk/src/main/java/cn/thinktoomuch/middleware/sdk/infrastructure/ai/dto/DeepseekChatCompletionResponseDTO.java
@@ -0,0 +1,45 @@
+package cn.thinktoomuch.middleware.sdk.infrastructure.ai.dto;
+
+import java.util.List;
+
+public class DeepseekChatCompletionResponseDTO {
+    private List<Message> message;
+
+    public static class Message {
+        private String role;
+        private String content;
+
+        public Message() {
+        }
+
+        public Message(String role, String content) {
+            this.role = role;
+            this.content = content;
+        }
+
+        public String getRole() {
+            return role;
+        }
+
+        public void setRole(String role) {
+            this.role = role;
+        }
+
+        public String getContent() {
+            return content;
+        }
+
+        public void setContent(String content) {
+            this.content = content;
+        }
+
+    }
+
+    public List<Message> getMessage() {
+        return message;
+    }
+
+    public void setMessage(List<Message> messages) {
+        this.message = messages;
+    }
+}
diff --git a/ai-code-review-sdk/src/main/java/cn/thinktoomuch/middleware/sdk/infrastructure/ai/dto/OpenAiChatCompletionResponseDTO.java b/ai-code-review-sdk/src/main/java/cn/thinktoomuch/middleware/sdk/infrastructure/ai/dto/OpenAiChatCompletionResponseDTO.java
new file mode 100644
index 0000000..4437261
--- /dev/null
+++ b/ai-code-review-sdk/src/main/java/cn/thinktoomuch/middleware/sdk/infrastructure/ai/dto/OpenAiChatCompletionResponseDTO.java
@@ -0,0 +1,48 @@
+package cn.thinktoomuch.middleware.sdk.infrastructure.ai.dto;
+
+import java.util.List;
+
+public class OpenAiChatCompletionResponseDTO {
+    private List<Choice> choices;
+
+    public static class Choice {
+        private Message message;
+
+        public Message getMessage() {
+            return message;
+        }
+
+        public void setMessage(Message message) {
+            this.message = message;
+        }
+    }
+
+    public static class Message {
+        private String role;
+        private String content;
+
+        public String getRole() {
+            return role;
+        }
+
+        public void setRole(String role) {
+            this.role = role;
+        }
+
+        public String getContent() {
+            return content;
+        }
+
+        public void setContent(String content) {
+            this.content = content;
+        }
+    }
+
+    public List<Choice> getChoices() {
+        return choices;
+    }
+
+    public void setChoices(List<Choice> choices) {
+        this.choices = choices;
+    }
+}
diff --git a/ai-code-review-sdk/src/main/java/cn/thinktoomuch/middleware/sdk/types/util/AIApiClientUtil.java b/ai-code-review-sdk/src/main/java/cn/thinktoomuch/middleware/sdk/types/util/AIApiClientUtil.java
index aaee4c1..837ac26 100644
--- a/ai-code-review-sdk/src/main/java/cn/thinktoomuch/middleware/sdk/types/util/AIApiClientUtil.java
+++ b/ai-code-review-sdk/src/main/java/cn/thinktoomuch/middleware/sdk/types/util/AIApiClientUtil.java
@@ -1,7 +1,8 @@
 package cn.thinktoomuch.middleware.sdk.types.util;
 
-import cn.thinktoomuch.middleware.sdk.domain.model.DeepSeekChatCompletionRequest;
-import cn.thinktoomuch.middleware.sdk.domain.model.OpenAIChatCompletionRequest;
+import cn.thinktoomuch.middleware.sdk.infrastructure.ai.dto.ChatCompletionRequestDTO;
+import cn.thinktoomuch.middleware.sdk.infrastructure.ai.dto.DeepseekChatCompletionResponseDTO;
+import cn.thinktoomuch.middleware.sdk.infrastructure.ai.dto.OpenAiChatCompletionResponseDTO;
 import com.alibaba.fastjson2.JSON;
 
 import java.io.*;
@@ -22,7 +23,7 @@ public class AIApiClientUtil {
      * @param completionRequest OpenAI请求对象
      * @return AI响应内容
      */
-    public static String callOpenAI(String apiKey, OpenAIChatCompletionRequest completionRequest) {
+    public static OpenAiChatCompletionResponseDTO callOpenAI(String apiKey, ChatCompletionRequestDTO completionRequest) {
         HttpURLConnection connection = null;
         try {
             // 1.创建URL对象
@@ -42,9 +43,10 @@ public class AIApiClientUtil {
             sendRequest(connection, requestBody);
 
             // 6. 读取响应
-            String responseBody = readResponse(connection);
+            String responseJSON = readResponse(connection);
+            OpenAiChatCompletionResponseDTO response = JSON.parseObject(responseJSON, OpenAiChatCompletionResponseDTO.class);
 
-            return parseOpenAIResponse(responseBody);
+            return response;
 
 
         } catch (Exception e) {
@@ -63,7 +65,7 @@ public class AIApiClientUtil {
      * @param completionRequest DeepSeek请求对象
      * @return AI响应内容
      */
-    public static String callDeepSeekOllama(String ollamaUrl, DeepSeekChatCompletionRequest completionRequest) {
+    public static DeepseekChatCompletionResponseDTO callDeepSeekOllama(String ollamaUrl, ChatCompletionRequestDTO completionRequest) {
         HttpURLConnection connection = null;
         try {
             // 1. 构建完整的API URL
@@ -83,10 +85,11 @@ public class AIApiClientUtil {
             sendRequest(connection, requestBody);
 
             // 6. 读取响应
-            String responseBody = readResponse(connection);
+            String responseJSON = readResponse(connection);
+            DeepseekChatCompletionResponseDTO response = JSON.parseObject(responseJSON, DeepseekChatCompletionResponseDTO.class);
 
             // 7. 解析响应
-            return parseOllamaResponse(responseBody);
+            return response;
 
         } catch (Exception e) {
             throw new RuntimeException("Ollama API调用失败", e);
@@ -188,6 +191,7 @@ public class AIApiClientUtil {
             }
         }
 
+        connection.disconnect();
         String response = responseBody.toString();
 //        System.out.println("响应内容: " + response);
 
diff --git a/ai-code-review-sdk/src/main/java/cn/thinktoomuch/middleware/sdk/types/util/RandomStringUtils.java b/ai-code-review-sdk/src/main/java/cn/thinktoomuch/middleware/sdk/types/util/RandomStringUtils.java
new file mode 100644
index 0000000..0328e4f
--- /dev/null
+++ b/ai-code-review-sdk/src/main/java/cn/thinktoomuch/middleware/sdk/types/util/RandomStringUtils.java
@@ -0,0 +1,17 @@
+package cn.thinktoomuch.middleware.sdk.types.util;
+
+import java.util.Random;
+
+public class RandomStringUtils {
+
+    public static String randomNumeric(int length) {
+        String characters = "ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789";
+        Random random = new Random();
+        StringBuilder sb = new StringBuilder(length);
+        for (int i = 0; i < length; i++) {
+            sb.append(characters.charAt(random.nextInt(characters.length())));
+        }
+        return sb.toString();
+    }
+
+}
diff --git a/ai-code-review-sdk/src/test/java/cn/thinktoomuch/middleware/sdk/test/ApiTest.java b/ai-code-review-sdk/src/test/java/cn/thinktoomuch/middleware/sdk/test/ApiTest.java
index 9f76592..82461bf 100644
--- a/ai-code-review-sdk/src/test/java/cn/thinktoomuch/middleware/sdk/test/ApiTest.java
+++ b/ai-code-review-sdk/src/test/java/cn/thinktoomuch/middleware/sdk/test/ApiTest.java
@@ -1,6 +1,9 @@
 package cn.thinktoomuch.middleware.sdk.test;
 
 import cn.thinktoomuch.middleware.sdk.domain.model.*;
+import cn.thinktoomuch.middleware.sdk.infrastructure.ai.dto.ChatCompletionRequestDTO;
+import cn.thinktoomuch.middleware.sdk.infrastructure.ai.dto.DeepseekChatCompletionResponseDTO;
+import cn.thinktoomuch.middleware.sdk.infrastructure.ai.dto.OpenAiChatCompletionResponseDTO;
 import cn.thinktoomuch.middleware.sdk.types.util.AIApiClientUtil;
 import cn.thinktoomuch.middleware.sdk.types.util.WeiXinAccessTokenUtil;
 import com.alibaba.fastjson2.JSON;
@@ -46,9 +49,9 @@ public class ApiTest {
                 + "请用中文回复。";
         testConnections(client);
 
-        demonstrateOpenAI(client, diffCode, systemMessage);
+//        demonstrateOpenAI(client, diffCode, systemMessage);
 
-//        demonstrateOllama(client, diffCode, systemMessage);
+        demonstrateOllama(client, diffCode, systemMessage);
 
 
     }
@@ -62,23 +65,24 @@ public class ApiTest {
 
         String apiKey = "sk-tCrJBw8G3UZm471ACqXMnlws4ct8UjBPUgXSSzW24M9CzYHq";
 
-        OpenAIChatCompletionRequest openAiRequest = new OpenAIChatCompletionRequest();
+        ChatCompletionRequestDTO openAiRequest = new ChatCompletionRequestDTO();
         openAiRequest.setModel(Model.GPT_4O_MINI.getCode());
-        openAiRequest.setMessages(new ArrayList<ChatCompletionRequest.Prompt>() {
+        openAiRequest.setMessages(new ArrayList<ChatCompletionRequestDTO.Prompt>() {
             private static final long serialVersionUID = -7988151926241837899L;
             {
-                add(new ChatCompletionRequest.Prompt("system", systemMessage));
-                add(new ChatCompletionRequest.Prompt("user", diffCode));
+                add(new ChatCompletionRequestDTO.Prompt("system", systemMessage));
+                add(new ChatCompletionRequestDTO.Prompt("user", diffCode));
             }
         });
-
+        openAiRequest.setStream(false);
         try {
             System.out.println("📡 正在调用OpenAI API...");
-            String result = client.callOpenAI(apiKey, openAiRequest);
+            OpenAiChatCompletionResponseDTO openAiReviewResult = client.callOpenAI(apiKey, openAiRequest);
+            OpenAiChatCompletionResponseDTO.Message message = openAiReviewResult.getChoices().get(0).getMessage();
 
             System.out.println("✅ OpenAI响应:");
             System.out.println("---");
-            System.out.println(result);
+            System.out.println(message.getContent());
             System.out.println("---");
 
         } catch (Exception e) {
@@ -104,21 +108,23 @@ public class ApiTest {
             ollamaUrl = "http://124.71.221.21:11434"; // 默认地址
         }
 
-        DeepSeekChatCompletionRequest deepSeekRequest = new DeepSeekChatCompletionRequest();
+        ChatCompletionRequestDTO deepSeekRequest = new ChatCompletionRequestDTO();
         deepSeekRequest.setModel(Model.DEEPSEEK_R1_1_5B.getCode());
-        deepSeekRequest.setMessages(new ArrayList<ChatCompletionRequest.Prompt>() {
+        deepSeekRequest.setMessages(new ArrayList<ChatCompletionRequestDTO.Prompt>() {
             private static final long serialVersionUID = -7988151926241837899L;
             {
-                add(new ChatCompletionRequest.Prompt("system", systemMessage));
-                add(new ChatCompletionRequest.Prompt("user", diffCode.toString()));
+                add(new ChatCompletionRequestDTO.Prompt("system", systemMessage));
+                add(new ChatCompletionRequestDTO.Prompt("user", diffCode.toString()));
             }
         });
+        deepSeekRequest.setStream(false);
 
         try {
             System.out.println("📡 正在调用Ollama API...");
             System.out.println("服务地址: " + ollamaUrl);
 
-            String result = client.callDeepSeekOllama(ollamaUrl, deepSeekRequest);
+            DeepseekChatCompletionResponseDTO deepSeekReviewResult = client.callDeepSeekOllama(ollamaUrl, deepSeekRequest);
+            String result = deepSeekReviewResult.getMessage().get(0).getContent();
 
             System.out.println("✅ Ollama响应:");
             System.out.println("---");
@@ -126,6 +132,7 @@ public class ApiTest {
             System.out.println("---");
 
         } catch (Exception e) {
+            e.printStackTrace();
             System.out.println("❌ Ollama调用失败: " + e.getMessage());
             System.out.println("💡 请确保Ollama服务正在运行:");
             System.out.println("   ollama serve");
```
