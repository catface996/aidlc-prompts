# 安全测试最佳实践

## 角色设定

你是一位精通安全测试的安全专家，擅长 OWASP、渗透测试、漏洞扫描和安全代码审计。

## 提示词模板

### 安全测试设计

```
请帮我设计安全测试方案：
- 测试目标：[Web应用/API/移动端]
- 测试范围：[认证/授权/数据/传输]
- 测试类型：[渗透测试/漏洞扫描/代码审计]
- 参考标准：[OWASP Top 10/等保]

请提供测试用例和自动化脚本。
```

## 核心代码示例

### OWASP ZAP 自动化扫描

```python
# security_scan.py
from zapv2 import ZAPv2
import time

class SecurityScanner:
    def __init__(self, target_url, zap_proxy='http://localhost:8080'):
        self.target = target_url
        self.zap = ZAPv2(proxies={'http': zap_proxy, 'https': zap_proxy})

    def spider_scan(self):
        """爬取目标网站"""
        print(f'Spidering target: {self.target}')
        scan_id = self.zap.spider.scan(self.target)

        while int(self.zap.spider.status(scan_id)) < 100:
            print(f'Spider progress: {self.zap.spider.status(scan_id)}%')
            time.sleep(5)

        print('Spider completed')
        return self.zap.spider.results(scan_id)

    def active_scan(self):
        """主动扫描"""
        print(f'Active scanning: {self.target}')
        scan_id = self.zap.ascan.scan(self.target)

        while int(self.zap.ascan.status(scan_id)) < 100:
            print(f'Scan progress: {self.zap.ascan.status(scan_id)}%')
            time.sleep(10)

        print('Active scan completed')

    def get_alerts(self):
        """获取安全告警"""
        alerts = self.zap.core.alerts(baseurl=self.target)
        return self._categorize_alerts(alerts)

    def _categorize_alerts(self, alerts):
        """按风险等级分类"""
        categorized = {'High': [], 'Medium': [], 'Low': [], 'Informational': []}
        for alert in alerts:
            risk = alert.get('risk', 'Informational')
            categorized[risk].append({
                'name': alert['name'],
                'url': alert['url'],
                'description': alert['description'],
                'solution': alert['solution']
            })
        return categorized

    def generate_report(self, output_file):
        """生成 HTML 报告"""
        report = self.zap.core.htmlreport()
        with open(output_file, 'w') as f:
            f.write(report)
        print(f'Report saved to: {output_file}')

# 使用示例
scanner = SecurityScanner('http://localhost:3000')
scanner.spider_scan()
scanner.active_scan()
alerts = scanner.get_alerts()
scanner.generate_report('security-report.html')
```

### SQL 注入测试

```java
// SqlInjectionTest.java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
class SqlInjectionTest {

    @Autowired
    private MockMvc mockMvc;

    @ParameterizedTest
    @DisplayName("SQL 注入测试")
    @ValueSource(strings = {
        "1 OR 1=1",
        "1; DROP TABLE users;--",
        "1' OR '1'='1",
        "1 UNION SELECT * FROM users--",
        "admin'--",
        "1; SELECT * FROM information_schema.tables--"
    })
    void shouldPreventSqlInjection(String maliciousInput) throws Exception {
        mockMvc.perform(get("/api/users")
                .param("id", maliciousInput))
            .andExpect(status().isBadRequest());
    }

    @Test
    @DisplayName("搜索功能 SQL 注入测试")
    void shouldPreventSqlInjectionInSearch() throws Exception {
        String[] payloads = {
            "test' OR '1'='1",
            "test%' OR 1=1--",
            "test'); DROP TABLE products;--"
        };

        for (String payload : payloads) {
            mockMvc.perform(get("/api/products/search")
                    .param("keyword", payload))
                .andExpect(result -> {
                    // 不应该返回所有数据
                    String content = result.getResponse().getContentAsString();
                    // 验证返回的不是全量数据
                });
        }
    }
}
```

### XSS 测试

```java
// XssTest.java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
class XssTest {

    @Autowired
    private MockMvc mockMvc;

    @ParameterizedTest
    @DisplayName("XSS 攻击测试")
    @ValueSource(strings = {
        "<script>alert('XSS')</script>",
        "<img src=x onerror=alert('XSS')>",
        "<svg onload=alert('XSS')>",
        "javascript:alert('XSS')",
        "<body onload=alert('XSS')>",
        "<iframe src='javascript:alert(1)'>",
        "'\"><script>alert(1)</script>",
        "<input onfocus=alert(1) autofocus>"
    })
    void shouldPreventXss(String maliciousInput) throws Exception {
        // 测试存储型 XSS
        mockMvc.perform(post("/api/comments")
                .contentType(MediaType.APPLICATION_JSON)
                .content("{\"content\":\"" + escapeJson(maliciousInput) + "\"}"))
            .andExpect(status().isOk());

        // 验证输出被转义
        mockMvc.perform(get("/api/comments/1"))
            .andExpect(result -> {
                String response = result.getResponse().getContentAsString();
                assertThat(response).doesNotContain("<script>");
                assertThat(response).doesNotContain("javascript:");
                assertThat(response).doesNotContain("onerror=");
            });
    }

    @Test
    @DisplayName("反射型 XSS 测试")
    void shouldPreventReflectedXss() throws Exception {
        String xssPayload = "<script>alert('XSS')</script>";

        mockMvc.perform(get("/api/search")
                .param("q", xssPayload))
            .andExpect(result -> {
                String response = result.getResponse().getContentAsString();
                assertThat(response).doesNotContain("<script>");
            });
    }
}
```

### 认证安全测试

```java
// AuthenticationSecurityTest.java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
class AuthenticationSecurityTest {

    @Autowired
    private MockMvc mockMvc;

    @Test
    @DisplayName("暴力破解防护测试")
    void shouldPreventBruteForceAttack() throws Exception {
        String username = "testuser";

        // 连续尝试错误密码
        for (int i = 0; i < 5; i++) {
            mockMvc.perform(post("/api/auth/login")
                    .contentType(MediaType.APPLICATION_JSON)
                    .content("{\"username\":\"" + username + "\",\"password\":\"wrong" + i + "\"}"})
                .andExpect(status().isUnauthorized());
        }

        // 第6次应该被锁定
        mockMvc.perform(post("/api/auth/login")
                .contentType(MediaType.APPLICATION_JSON)
                .content("{\"username\":\"" + username + "\",\"password\":\"wrong\"}"))
            .andExpect(status().isTooManyRequests())
            .andExpect(jsonPath("$.message").value(containsString("locked")));
    }

    @Test
    @DisplayName("会话固定攻击防护")
    void shouldPreventSessionFixation() throws Exception {
        // 获取登录前的 Session ID
        MvcResult beforeLogin = mockMvc.perform(get("/api/users/me"))
            .andReturn();
        String sessionBefore = beforeLogin.getRequest().getSession().getId();

        // 登录
        MvcResult afterLogin = mockMvc.perform(post("/api/auth/login")
                .contentType(MediaType.APPLICATION_JSON)
                .content("{\"username\":\"testuser\",\"password\":\"password\"}"))
            .andReturn();
        String sessionAfter = afterLogin.getRequest().getSession().getId();

        // Session ID 应该改变
        assertThat(sessionAfter).isNotEqualTo(sessionBefore);
    }

    @Test
    @DisplayName("JWT Token 安全测试")
    void shouldValidateJwtSecurity() throws Exception {
        // 测试过期 Token
        String expiredToken = generateExpiredToken();
        mockMvc.perform(get("/api/users/me")
                .header("Authorization", "Bearer " + expiredToken))
            .andExpect(status().isUnauthorized());

        // 测试篡改 Token
        String tamperedToken = "eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiJhZG1pbiJ9.tampered";
        mockMvc.perform(get("/api/users/me")
                .header("Authorization", "Bearer " + tamperedToken))
            .andExpect(status().isUnauthorized());

        // 测试无签名 Token
        String unsignedToken = "eyJhbGciOiJub25lIn0.eyJzdWIiOiJhZG1pbiJ9.";
        mockMvc.perform(get("/api/users/me")
                .header("Authorization", "Bearer " + unsignedToken))
            .andExpect(status().isUnauthorized());
    }

    @Test
    @DisplayName("密码策略测试")
    void shouldEnforcePasswordPolicy() throws Exception {
        // 弱密码应被拒绝
        String[] weakPasswords = {"123456", "password", "qwerty", "abc123", "12345678"};

        for (String weakPassword : weakPasswords) {
            mockMvc.perform(post("/api/users/register")
                    .contentType(MediaType.APPLICATION_JSON)
                    .content("{\"username\":\"newuser\",\"password\":\"" + weakPassword + "\"}"))
                .andExpect(status().isBadRequest())
                .andExpect(jsonPath("$.errors[*].field").value(hasItem("password")));
        }
    }
}
```

### 授权测试

```java
// AuthorizationTest.java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
class AuthorizationTest {

    @Autowired
    private MockMvc mockMvc;

    @Test
    @DisplayName("越权访问测试 - 水平越权")
    void shouldPreventHorizontalPrivilegeEscalation() throws Exception {
        // 用户 A 的 Token
        String userAToken = loginAndGetToken("userA", "password");
        // 用户 B 的订单
        Long userBOrderId = 100L;

        // 用户 A 不能访问用户 B 的订单
        mockMvc.perform(get("/api/orders/" + userBOrderId)
                .header("Authorization", "Bearer " + userAToken))
            .andExpect(status().isForbidden());
    }

    @Test
    @DisplayName("越权访问测试 - 垂直越权")
    void shouldPreventVerticalPrivilegeEscalation() throws Exception {
        // 普通用户 Token
        String userToken = loginAndGetToken("user", "password");

        // 普通用户不能访问管理员接口
        mockMvc.perform(get("/api/admin/users")
                .header("Authorization", "Bearer " + userToken))
            .andExpect(status().isForbidden());

        mockMvc.perform(delete("/api/admin/users/1")
                .header("Authorization", "Bearer " + userToken))
            .andExpect(status().isForbidden());
    }

    @Test
    @DisplayName("IDOR 漏洞测试")
    void shouldPreventInsecureDirectObjectReference() throws Exception {
        String userToken = loginAndGetToken("user", "password");

        // 尝试通过修改 ID 访问其他用户数据
        for (long id = 1; id <= 10; id++) {
            mockMvc.perform(get("/api/users/" + id + "/profile")
                    .header("Authorization", "Bearer " + userToken))
                .andExpect(result -> {
                    if (result.getResponse().getStatus() == 200) {
                        // 只能访问自己的数据
                        String response = result.getResponse().getContentAsString();
                        // 验证返回的是当前用户的数据
                    }
                });
        }
    }
}
```

### 敏感数据测试

```java
// SensitiveDataTest.java
@SpringBootTest
class SensitiveDataTest {

    @Test
    @DisplayName("敏感数据不应出现在日志中")
    void shouldNotLogSensitiveData() throws Exception {
        // 读取日志文件
        String logContent = Files.readString(Path.of("logs/application.log"));

        // 不应包含密码
        assertThat(logContent).doesNotContain("password123");
        assertThat(logContent).doesNotContain("secret");

        // 不应包含完整的信用卡号
        assertThat(logContent).doesNotMatch(".*\\d{16}.*");

        // 不应包含身份证号
        assertThat(logContent).doesNotMatch(".*\\d{18}.*");
    }

    @Test
    @DisplayName("API 响应不应包含敏感字段")
    void shouldNotExposeSensitiveFieldsInResponse() throws Exception {
        mockMvc.perform(get("/api/users/1")
                .header("Authorization", "Bearer " + validToken))
            .andExpect(jsonPath("$.password").doesNotExist())
            .andExpect(jsonPath("$.passwordHash").doesNotExist())
            .andExpect(jsonPath("$.salt").doesNotExist())
            .andExpect(jsonPath("$.secretKey").doesNotExist());
    }
}
```

## OWASP Top 10 测试清单

| 风险 | 测试项 |
|------|--------|
| A01 访问控制失效 | 越权访问、IDOR |
| A02 加密失败 | 敏感数据加密、传输加密 |
| A03 注入 | SQL注入、命令注入、XSS |
| A04 不安全设计 | 业务逻辑漏洞 |
| A05 安全配置错误 | 默认配置、错误信息泄露 |
| A06 过时组件 | 依赖漏洞扫描 |
| A07 认证失败 | 暴力破解、会话管理 |
| A08 软件完整性 | 依赖检查、CI/CD 安全 |
| A09 日志监控失败 | 日志记录、告警机制 |
| A10 SSRF | 服务端请求伪造 |

## 最佳实践清单

- [ ] SQL 注入测试
- [ ] XSS 测试 (存储/反射/DOM)
- [ ] 认证安全测试
- [ ] 授权和越权测试
- [ ] 敏感数据保护测试
- [ ] 安全配置检查
- [ ] 依赖漏洞扫描
- [ ] 定期渗透测试
