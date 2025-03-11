根据提供的`git diff`记录，以下是对代码变更的评审：

**ChatGLM.java**

- **变更描述**: `ChatGLM` 类的异常处理从包含异常消息的日志输出，变为仅输出异常对象。

**评审意见**:
1. **改进**: 删除异常消息的具体内容是一种安全实践，因为这可能暴露敏感信息。
2. **风险**: 虽然这是一个改进，但没有提供任何关于异常原因的上下文信息，这可能导致调试困难。
3. **建议**: 建议保留异常消息，并确保在日志中记录足够的信息，以便在需要时进行调试，但仍然考虑移除敏感信息。

**HttpUtils.java**

- **变更描述**: `HttpUtils` 类的 `parseObject` 方法中添加了异常处理，并且使用了`Assert.thrown`抛出异常。

**评审意见**:
1. **改进**: 添加异常处理是一个很好的实践，它能够处理解析JSON时可能发生的错误。
2. **风险**: 使用 `Assert.thrown` 可能不是最佳做法，因为它会中断程序流程并抛出一个异常，这可能会导致调用代码难以处理这种情况。
3. **建议**: 建议使用一个返回值来表示解析结果和异常情况，而不是抛出异常。这样，调用者可以根据返回值来判断是否发生错误，并相应地处理。
4. **代码示例**:
   ```java
   public static <T> T parseObject(String content, Class<T> result) {
       try {
           return JSON.parseObject(content, result);
       } catch (Exception e) {
           logger.error("parseObject error: {}", content);
           // 处理异常，例如返回一个错误信息或特殊对象
           return null; // 或者抛出自定义异常，但确保调用者知道如何处理它
       }
   }
   ```
5. **其他**:
   - 建议检查`HttpUtils`类中其他使用`connection.getResponseCode()`的地方，以确保它们也进行了适当的异常处理。
   - 确保`HttpUtils`类中使用了合适的日志级别，对于错误处理级别的日志（如`ERROR`），确保它们被正确记录，而不是被静默处理。