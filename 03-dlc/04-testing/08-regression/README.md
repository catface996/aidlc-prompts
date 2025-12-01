# 回归测试最佳实践

## 角色设定

你是一位精通回归测试的质量保证专家，擅长测试策略制定、自动化回归套件设计和持续集成中的回归测试优化。

## 提示词模板

### 回归测试设计

```
请帮我设计回归测试方案：
- 变更范围：[功能变更/Bug修复/重构]
- 影响分析：[受影响的模块和功能]
- 测试策略：[完整回归/选择性回归/风险回归]
- 自动化程度：[自动化比例要求]

请提供测试用例选择策略和执行计划。
```

## 核心代码示例

### 基于变更的回归测试选择

```java
// RegressionTestSelector.java
@Component
public class RegressionTestSelector {

    private final TestCaseRepository testCaseRepository;
    private final CodeCoverageAnalyzer coverageAnalyzer;
    private final GitClient gitClient;

    public RegressionTestSelector(TestCaseRepository testCaseRepository,
                                   CodeCoverageAnalyzer coverageAnalyzer,
                                   GitClient gitClient) {
        this.testCaseRepository = testCaseRepository;
        this.coverageAnalyzer = coverageAnalyzer;
        this.gitClient = gitClient;
    }

    /**
     * 基于代码变更选择回归测试用例
     */
    public List<TestCase> selectTestCases(String fromCommit, String toCommit) {
        // 1. 获取变更的文件
        Set<String> changedFiles = gitClient.getChangedFiles(fromCommit, toCommit);

        // 2. 分析变更影响
        Set<String> affectedClasses = analyzeAffectedClasses(changedFiles);

        // 3. 获取覆盖受影响类的测试用例
        Set<TestCase> selectedTests = new HashSet<>();

        for (String affectedClass : affectedClasses) {
            List<TestCase> coveringTests = coverageAnalyzer
                .getTestsCoveringClass(affectedClass);
            selectedTests.addAll(coveringTests);
        }

        // 4. 添加核心业务流程测试
        selectedTests.addAll(testCaseRepository.findByPriority(Priority.CRITICAL));

        // 5. 添加最近失败过的测试
        selectedTests.addAll(testCaseRepository.findRecentlyFailed(Duration.ofDays(7)));

        return new ArrayList<>(selectedTests);
    }

    /**
     * 分析变更影响的类
     */
    private Set<String> analyzeAffectedClasses(Set<String> changedFiles) {
        Set<String> affected = new HashSet<>();

        for (String file : changedFiles) {
            // 直接变更的类
            String className = fileToClassName(file);
            affected.add(className);

            // 依赖该类的其他类
            Set<String> dependents = coverageAnalyzer.getDependentClasses(className);
            affected.addAll(dependents);
        }

        return affected;
    }

    private String fileToClassName(String filePath) {
        return filePath
            .replace("src/main/java/", "")
            .replace("src/test/java/", "")
            .replace("/", ".")
            .replace(".java", "");
    }
}
```

### 回归测试套件管理

```java
// RegressionTestSuite.java
@ExtendWith(SpringExtension.class)
@SpringBootTest
@Tag("regression")
public class RegressionTestSuite {

    @Nested
    @DisplayName("核心业务流程回归")
    @Tag("critical")
    class CriticalBusinessFlows {

        @Test
        @DisplayName("用户注册流程")
        void userRegistrationFlow() {
            // 测试完整的用户注册流程
        }

        @Test
        @DisplayName("订单创建流程")
        void orderCreationFlow() {
            // 测试完整的订单创建流程
        }

        @Test
        @DisplayName("支付流程")
        void paymentFlow() {
            // 测试完整的支付流程
        }

        @Test
        @DisplayName("退款流程")
        void refundFlow() {
            // 测试完整的退款流程
        }
    }

    @Nested
    @DisplayName("用户模块回归")
    @Tag("user-module")
    class UserModuleRegression {

        @Test
        @DisplayName("用户登录")
        void userLogin() {
            // 测试用户登录功能
        }

        @Test
        @DisplayName("密码重置")
        void passwordReset() {
            // 测试密码重置功能
        }

        @Test
        @DisplayName("用户信息更新")
        void userProfileUpdate() {
            // 测试用户信息更新
        }
    }

    @Nested
    @DisplayName("订单模块回归")
    @Tag("order-module")
    class OrderModuleRegression {

        @Test
        @DisplayName("订单查询")
        void orderQuery() {
            // 测试订单查询
        }

        @Test
        @DisplayName("订单状态变更")
        void orderStatusChange() {
            // 测试订单状态变更
        }

        @Test
        @DisplayName("订单取消")
        void orderCancellation() {
            // 测试订单取消
        }
    }
}

// 按标签运行测试
// mvn test -Dgroups="critical"
// mvn test -Dgroups="user-module,order-module"
```

### 智能回归测试优先级

```java
// TestPrioritizer.java
@Component
public class TestPrioritizer {

    private final TestHistoryRepository testHistory;
    private final CodeChangeAnalyzer changeAnalyzer;

    /**
     * 计算测试用例优先级分数
     */
    public double calculatePriority(TestCase testCase, ChangeContext context) {
        double score = 0.0;

        // 1. 历史失败率 (权重: 30%)
        double failureRate = testHistory.getFailureRate(testCase.getId());
        score += failureRate * 0.3;

        // 2. 最近执行时间 (权重: 20%)
        Duration sinceLastRun = testHistory.getTimeSinceLastRun(testCase.getId());
        score += calculateTimeScore(sinceLastRun) * 0.2;

        // 3. 代码变更相关性 (权重: 30%)
        double changeRelevance = changeAnalyzer.calculateRelevance(
            testCase, context.getChangedFiles());
        score += changeRelevance * 0.3;

        // 4. 业务重要性 (权重: 20%)
        score += testCase.getBusinessPriority().getScore() * 0.2;

        return score;
    }

    private double calculateTimeScore(Duration sinceLastRun) {
        long hours = sinceLastRun.toHours();
        if (hours < 24) return 0.2;
        if (hours < 72) return 0.5;
        if (hours < 168) return 0.8;
        return 1.0;
    }

    /**
     * 获取优先级排序的测试列表
     */
    public List<TestCase> prioritize(List<TestCase> tests, ChangeContext context) {
        return tests.stream()
            .map(test -> new PrioritizedTest(test, calculatePriority(test, context)))
            .sorted(Comparator.comparingDouble(PrioritizedTest::getScore).reversed())
            .map(PrioritizedTest::getTestCase)
            .collect(Collectors.toList());
    }
}
```

### 视觉回归测试

```typescript
// visual-regression.spec.ts
import { test, expect } from '@playwright/test';

test.describe('视觉回归测试', () => {
  test.beforeEach(async ({ page }) => {
    // 设置一致的视口大小
    await page.setViewportSize({ width: 1280, height: 720 });
    // 禁用动画以确保截图一致性
    await page.addStyleTag({
      content: `
        *, *::before, *::after {
          animation-duration: 0s !important;
          transition-duration: 0s !important;
        }
      `,
    });
  });

  test('首页视觉回归', async ({ page }) => {
    await page.goto('/');
    await page.waitForLoadState('networkidle');

    await expect(page).toHaveScreenshot('homepage.png', {
      maxDiffPixels: 100,
      threshold: 0.1,
    });
  });

  test('商品列表页视觉回归', async ({ page }) => {
    await page.goto('/products');
    await page.waitForSelector('[data-testid="product-grid"]');

    await expect(page).toHaveScreenshot('products-list.png', {
      mask: [page.locator('[data-testid="dynamic-price"]')],  // 遮盖动态内容
    });
  });

  test('商品详情页视觉回归', async ({ page }) => {
    await page.goto('/products/1');
    await page.waitForLoadState('networkidle');

    // 整页截图
    await expect(page).toHaveScreenshot('product-detail-full.png', {
      fullPage: true,
    });

    // 组件截图
    await expect(page.locator('[data-testid="product-gallery"]'))
      .toHaveScreenshot('product-gallery.png');

    await expect(page.locator('[data-testid="product-info"]'))
      .toHaveScreenshot('product-info.png');
  });

  test('购物车视觉回归', async ({ page }) => {
    // 添加商品到购物车
    await page.goto('/products/1');
    await page.click('[data-testid="add-to-cart"]');

    await page.goto('/cart');
    await page.waitForLoadState('networkidle');

    await expect(page).toHaveScreenshot('cart-with-item.png');
  });

  test('响应式视觉回归', async ({ page }) => {
    const viewports = [
      { width: 1920, height: 1080, name: 'desktop-large' },
      { width: 1366, height: 768, name: 'desktop' },
      { width: 768, height: 1024, name: 'tablet' },
      { width: 375, height: 667, name: 'mobile' },
    ];

    for (const viewport of viewports) {
      await page.setViewportSize({ width: viewport.width, height: viewport.height });
      await page.goto('/');
      await page.waitForLoadState('networkidle');

      await expect(page).toHaveScreenshot(`homepage-${viewport.name}.png`);
    }
  });
});

// 更新基线截图的命令
// npx playwright test --update-snapshots
```

### 数据驱动回归测试

```java
// DataDrivenRegressionTest.java
@SpringBootTest
class DataDrivenRegressionTest {

    @Autowired
    private OrderService orderService;

    @ParameterizedTest
    @DisplayName("订单创建回归测试")
    @CsvFileSource(resources = "/regression-data/order-creation.csv", numLinesToSkip = 1)
    void shouldCreateOrderCorrectly(
            Long userId,
            Long productId,
            int quantity,
            String expectedStatus,
            BigDecimal expectedAmount) {

        OrderRequest request = OrderRequest.builder()
            .userId(userId)
            .productId(productId)
            .quantity(quantity)
            .build();

        OrderResponse response = orderService.createOrder(request);

        assertThat(response.getStatus()).isEqualTo(expectedStatus);
        assertThat(response.getTotalAmount()).isEqualByComparingTo(expectedAmount);
    }

    @ParameterizedTest
    @DisplayName("价格计算回归测试")
    @MethodSource("priceCalculationTestCases")
    void shouldCalculatePriceCorrectly(PriceTestCase testCase) {
        BigDecimal result = orderService.calculatePrice(
            testCase.getBasePrice(),
            testCase.getQuantity(),
            testCase.getDiscountCode()
        );

        assertThat(result).isEqualByComparingTo(testCase.getExpectedPrice());
    }

    static Stream<PriceTestCase> priceCalculationTestCases() {
        return Stream.of(
            new PriceTestCase(new BigDecimal("100"), 1, null, new BigDecimal("100")),
            new PriceTestCase(new BigDecimal("100"), 2, null, new BigDecimal("200")),
            new PriceTestCase(new BigDecimal("100"), 1, "SAVE10", new BigDecimal("90")),
            new PriceTestCase(new BigDecimal("100"), 3, "BULK20", new BigDecimal("240"))
        );
    }
}

// regression-data/order-creation.csv
// userId,productId,quantity,expectedStatus,expectedAmount
// 1,100,1,CREATED,99.99
// 1,100,2,CREATED,199.98
// 1,101,1,CREATED,149.99
// 2,100,5,CREATED,499.95
```

### 回归测试报告生成

```java
// RegressionReportGenerator.java
@Component
public class RegressionReportGenerator {

    public RegressionReport generateReport(List<TestResult> results,
                                            ChangeContext context) {
        RegressionReport report = new RegressionReport();

        // 基本统计
        report.setTotalTests(results.size());
        report.setPassedTests(countByStatus(results, TestStatus.PASSED));
        report.setFailedTests(countByStatus(results, TestStatus.FAILED));
        report.setSkippedTests(countByStatus(results, TestStatus.SKIPPED));

        // 计算通过率
        report.setPassRate(calculatePassRate(results));

        // 变更相关信息
        report.setChangedFiles(context.getChangedFiles());
        report.setCommitRange(context.getFromCommit() + ".." + context.getToCommit());

        // 失败测试详情
        report.setFailedTestDetails(results.stream()
            .filter(r -> r.getStatus() == TestStatus.FAILED)
            .map(this::toFailedTestDetail)
            .collect(Collectors.toList()));

        // 新增失败 (之前通过，现在失败)
        report.setNewFailures(findNewFailures(results));

        // 修复的测试 (之前失败，现在通过)
        report.setFixedTests(findFixedTests(results));

        // 执行时间分析
        report.setTotalDuration(calculateTotalDuration(results));
        report.setSlowestTests(findSlowestTests(results, 10));

        return report;
    }

    private FailedTestDetail toFailedTestDetail(TestResult result) {
        return FailedTestDetail.builder()
            .testName(result.getTestName())
            .className(result.getClassName())
            .errorMessage(result.getErrorMessage())
            .stackTrace(result.getStackTrace())
            .duration(result.getDuration())
            .relatedChanges(findRelatedChanges(result))
            .build();
    }
}
```

### CI/CD 回归测试配置

```yaml
# .github/workflows/regression.yml
name: Regression Tests

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  analyze-changes:
    runs-on: ubuntu-latest
    outputs:
      affected-modules: ${{ steps.analyze.outputs.modules }}
      test-scope: ${{ steps.analyze.outputs.scope }}
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Analyze changes
        id: analyze
        run: |
          # 分析变更的文件
          CHANGED_FILES=$(git diff --name-only ${{ github.event.before }} ${{ github.sha }})

          # 确定受影响的模块
          if echo "$CHANGED_FILES" | grep -q "src/main/java/.*user"; then
            echo "modules=user-module" >> $GITHUB_OUTPUT
          fi

          if echo "$CHANGED_FILES" | grep -q "src/main/java/.*order"; then
            echo "modules=order-module" >> $GITHUB_OUTPUT
          fi

          # 确定测试范围
          if echo "$CHANGED_FILES" | grep -q "core/"; then
            echo "scope=full" >> $GITHUB_OUTPUT
          else
            echo "scope=selective" >> $GITHUB_OUTPUT
          fi

  critical-regression:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Java
        uses: actions/setup-java@v4
        with:
          java-version: '17'
          distribution: 'temurin'

      - name: Run critical tests
        run: ./gradlew test -Dgroups="critical" --parallel

      - name: Upload results
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: critical-test-results
          path: build/reports/tests/

  selective-regression:
    needs: analyze-changes
    if: needs.analyze-changes.outputs.test-scope == 'selective'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Java
        uses: actions/setup-java@v4
        with:
          java-version: '17'
          distribution: 'temurin'

      - name: Run selective tests
        run: |
          MODULES="${{ needs.analyze-changes.outputs.affected-modules }}"
          ./gradlew test -Dgroups="$MODULES" --parallel

  full-regression:
    needs: analyze-changes
    if: needs.analyze-changes.outputs.test-scope == 'full'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Java
        uses: actions/setup-java@v4
        with:
          java-version: '17'
          distribution: 'temurin'

      - name: Run full regression
        run: ./gradlew test -Dgroups="regression" --parallel

  visual-regression:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'

      - name: Install dependencies
        run: npm ci

      - name: Install Playwright
        run: npx playwright install --with-deps

      - name: Run visual regression
        run: npx playwright test --grep "@visual"

      - name: Upload diff screenshots
        uses: actions/upload-artifact@v4
        if: failure()
        with:
          name: visual-diff
          path: test-results/

  regression-report:
    needs: [critical-regression, selective-regression, full-regression]
    if: always()
    runs-on: ubuntu-latest
    steps:
      - name: Download all artifacts
        uses: actions/download-artifact@v4

      - name: Generate report
        run: |
          # 合并测试结果并生成报告
          echo "Generating regression test report..."

      - name: Notify on failure
        if: failure()
        uses: slackapi/slack-github-action@v1
        with:
          payload: |
            {
              "text": "Regression tests failed for ${{ github.repository }}",
              "blocks": [
                {
                  "type": "section",
                  "text": {
                    "type": "mrkdwn",
                    "text": "*Regression Test Failure*\nRepository: ${{ github.repository }}\nBranch: ${{ github.ref }}\nCommit: ${{ github.sha }}"
                  }
                }
              ]
            }
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK }}
```

### 回归测试执行策略

```java
// RegressionTestExecutor.java
@Component
public class RegressionTestExecutor {

    private final TestPrioritizer prioritizer;
    private final RegressionTestSelector selector;
    private final TestRunner testRunner;

    /**
     * 执行回归测试
     */
    public RegressionTestResult execute(RegressionTestConfig config) {
        // 1. 选择测试用例
        List<TestCase> selectedTests = selector.selectTestCases(
            config.getFromCommit(),
            config.getToCommit()
        );

        // 2. 按优先级排序
        ChangeContext context = buildChangeContext(config);
        List<TestCase> prioritizedTests = prioritizer.prioritize(selectedTests, context);

        // 3. 根据时间预算裁剪
        List<TestCase> testsToRun = applyTimeBudget(
            prioritizedTests,
            config.getTimeBudgetMinutes()
        );

        // 4. 执行测试
        List<TestResult> results = testRunner.runTests(testsToRun, config.getParallelism());

        // 5. 快速失败检查
        if (config.isFailFast() && hasFailures(results)) {
            return RegressionTestResult.failed(results, "Fast fail triggered");
        }

        return RegressionTestResult.completed(results);
    }

    private List<TestCase> applyTimeBudget(List<TestCase> tests, int budgetMinutes) {
        if (budgetMinutes <= 0) {
            return tests;
        }

        List<TestCase> selected = new ArrayList<>();
        int totalEstimatedTime = 0;
        int budgetSeconds = budgetMinutes * 60;

        for (TestCase test : tests) {
            int estimatedTime = test.getEstimatedDurationSeconds();
            if (totalEstimatedTime + estimatedTime <= budgetSeconds) {
                selected.add(test);
                totalEstimatedTime += estimatedTime;
            }
        }

        return selected;
    }
}
```

## 回归测试策略

| 策略 | 适用场景 | 执行时间 | 覆盖范围 |
|------|----------|----------|----------|
| 完整回归 | 重大版本发布 | 长 | 100% |
| 选择性回归 | 日常迭代 | 中 | 受影响模块 |
| 冒烟回归 | 快速验证 | 短 | 核心流程 |
| 风险回归 | 高风险变更 | 中 | 高风险区域 |

## 回归测试指标

| 指标 | 说明 | 目标值 |
|------|------|--------|
| 回归通过率 | 通过测试/总测试 | > 99% |
| 新增失败率 | 新失败/总测试 | < 1% |
| 测试执行时间 | 完整回归耗时 | < 30分钟 |
| 测试选择精确度 | 选中失败测试/总失败 | > 90% |
| 代码覆盖率 | 回归测试覆盖的代码 | > 80% |

## 最佳实践清单

- [ ] 建立回归测试基线
- [ ] 实施测试用例优先级排序
- [ ] 基于变更选择测试用例
- [ ] 视觉回归测试覆盖核心页面
- [ ] 数据驱动测试覆盖边界场景
- [ ] 自动化生成回归报告
- [ ] CI/CD 集成回归测试
- [ ] 定期维护和清理测试套件
