# 单元测试最佳实践

## 角色设定

你是一位精通单元测试的软件质量专家，擅长 JUnit、Mockito、Jest 等测试框架，注重测试覆盖率和代码质量。

## 提示词模板

### 单元测试编写

```
请帮我编写单元测试：
- 被测代码：[粘贴代码或描述功能]
- 测试框架：[JUnit/Jest/Vitest/pytest]
- 覆盖场景：
  - [ ] 正常流程
  - [ ] 边界条件
  - [ ] 异常情况
  - [ ] 空值处理
- Mock 需求：[需要模拟的依赖]

请提供完整的测试代码。
```

### 测试覆盖率分析

```
请分析以下代码的测试覆盖情况：
[粘贴代码]

请识别：
1. 未覆盖的分支
2. 边界条件
3. 建议补充的测试用例
```

## 核心代码示例

### Java - JUnit 5 + Mockito

```java
// OrderServiceTest.java
@ExtendWith(MockitoExtension.class)
class OrderServiceTest {

    @Mock
    private OrderRepository orderRepository;

    @Mock
    private InventoryService inventoryService;

    @Mock
    private PaymentService paymentService;

    @InjectMocks
    private OrderService orderService;

    @Captor
    private ArgumentCaptor<Order> orderCaptor;

    @Nested
    @DisplayName("创建订单")
    class CreateOrder {

        @Test
        @DisplayName("成功创建订单")
        void shouldCreateOrderSuccessfully() {
            // Given
            OrderRequest request = OrderRequest.builder()
                .userId(1L)
                .productId(100L)
                .quantity(2)
                .build();

            when(inventoryService.checkStock(100L, 2)).thenReturn(true);
            when(inventoryService.deduct(100L, 2)).thenReturn(true);
            when(orderRepository.save(any(Order.class)))
                .thenAnswer(inv -> {
                    Order order = inv.getArgument(0);
                    order.setId(1L);
                    return order;
                });

            // When
            Order result = orderService.createOrder(request);

            // Then
            assertThat(result).isNotNull();
            assertThat(result.getId()).isEqualTo(1L);
            assertThat(result.getUserId()).isEqualTo(1L);
            assertThat(result.getStatus()).isEqualTo(OrderStatus.CREATED);

            verify(orderRepository).save(orderCaptor.capture());
            Order savedOrder = orderCaptor.getValue();
            assertThat(savedOrder.getQuantity()).isEqualTo(2);
        }

        @Test
        @DisplayName("库存不足时抛出异常")
        void shouldThrowExceptionWhenInsufficientStock() {
            // Given
            OrderRequest request = OrderRequest.builder()
                .userId(1L)
                .productId(100L)
                .quantity(100)
                .build();

            when(inventoryService.checkStock(100L, 100)).thenReturn(false);

            // When & Then
            assertThatThrownBy(() -> orderService.createOrder(request))
                .isInstanceOf(BusinessException.class)
                .hasFieldOrPropertyWithValue("errorCode", ErrorCode.INSUFFICIENT_STOCK);

            verify(orderRepository, never()).save(any());
        }

        @ParameterizedTest
        @DisplayName("验证订单数量边界")
        @ValueSource(ints = {0, -1, -100})
        void shouldRejectInvalidQuantity(int quantity) {
            OrderRequest request = OrderRequest.builder()
                .userId(1L)
                .productId(100L)
                .quantity(quantity)
                .build();

            assertThatThrownBy(() -> orderService.createOrder(request))
                .isInstanceOf(IllegalArgumentException.class);
        }

        @ParameterizedTest
        @DisplayName("验证不同商品的订单创建")
        @CsvSource({
            "1, 100, 2, CREATED",
            "2, 200, 5, CREATED",
            "3, 300, 1, CREATED"
        })
        void shouldCreateOrderForDifferentProducts(
                Long userId, Long productId, int quantity, OrderStatus expectedStatus) {

            OrderRequest request = OrderRequest.builder()
                .userId(userId)
                .productId(productId)
                .quantity(quantity)
                .build();

            when(inventoryService.checkStock(productId, quantity)).thenReturn(true);
            when(inventoryService.deduct(productId, quantity)).thenReturn(true);
            when(orderRepository.save(any())).thenAnswer(inv -> inv.getArgument(0));

            Order result = orderService.createOrder(request);

            assertThat(result.getStatus()).isEqualTo(expectedStatus);
        }
    }

    @Nested
    @DisplayName("取消订单")
    class CancelOrder {

        @Test
        @DisplayName("成功取消待支付订单")
        void shouldCancelPendingOrder() {
            // Given
            Order order = Order.builder()
                .id(1L)
                .status(OrderStatus.CREATED)
                .productId(100L)
                .quantity(2)
                .build();

            when(orderRepository.findById(1L)).thenReturn(Optional.of(order));
            when(inventoryService.restore(100L, 2)).thenReturn(true);

            // When
            orderService.cancelOrder(1L);

            // Then
            assertThat(order.getStatus()).isEqualTo(OrderStatus.CANCELLED);
            verify(orderRepository).save(order);
        }

        @Test
        @DisplayName("不能取消已完成的订单")
        void shouldNotCancelCompletedOrder() {
            Order order = Order.builder()
                .id(1L)
                .status(OrderStatus.COMPLETED)
                .build();

            when(orderRepository.findById(1L)).thenReturn(Optional.of(order));

            assertThatThrownBy(() -> orderService.cancelOrder(1L))
                .isInstanceOf(BusinessException.class)
                .hasFieldOrPropertyWithValue("errorCode", ErrorCode.ORDER_CANNOT_CANCEL);
        }
    }

    @Test
    @DisplayName("测试超时场景")
    @Timeout(value = 100, unit = TimeUnit.MILLISECONDS)
    void shouldCompleteWithinTimeout() {
        // 测试方法执行不超过 100ms
    }
}
```

### JavaScript/TypeScript - Jest

```typescript
// orderService.test.ts
import { OrderService } from './orderService';
import { OrderRepository } from './orderRepository';
import { InventoryService } from './inventoryService';

// Mock 模块
jest.mock('./orderRepository');
jest.mock('./inventoryService');

describe('OrderService', () => {
  let orderService: OrderService;
  let mockOrderRepository: jest.Mocked<OrderRepository>;
  let mockInventoryService: jest.Mocked<InventoryService>;

  beforeEach(() => {
    jest.clearAllMocks();
    mockOrderRepository = new OrderRepository() as jest.Mocked<OrderRepository>;
    mockInventoryService = new InventoryService() as jest.Mocked<InventoryService>;
    orderService = new OrderService(mockOrderRepository, mockInventoryService);
  });

  describe('createOrder', () => {
    it('should create order successfully', async () => {
      // Arrange
      const request = { userId: 1, productId: 100, quantity: 2 };
      mockInventoryService.checkStock.mockResolvedValue(true);
      mockInventoryService.deduct.mockResolvedValue(true);
      mockOrderRepository.save.mockResolvedValue({ id: 1, ...request, status: 'CREATED' });

      // Act
      const result = await orderService.createOrder(request);

      // Assert
      expect(result).toEqual(expect.objectContaining({
        id: 1,
        userId: 1,
        status: 'CREATED',
      }));
      expect(mockInventoryService.checkStock).toHaveBeenCalledWith(100, 2);
      expect(mockOrderRepository.save).toHaveBeenCalledTimes(1);
    });

    it('should throw error when stock is insufficient', async () => {
      const request = { userId: 1, productId: 100, quantity: 100 };
      mockInventoryService.checkStock.mockResolvedValue(false);

      await expect(orderService.createOrder(request))
        .rejects
        .toThrow('Insufficient stock');

      expect(mockOrderRepository.save).not.toHaveBeenCalled();
    });

    it.each([
      [0, 'Quantity must be positive'],
      [-1, 'Quantity must be positive'],
      [null, 'Quantity is required'],
    ])('should reject quantity %s with error: %s', async (quantity, expectedError) => {
      const request = { userId: 1, productId: 100, quantity };

      await expect(orderService.createOrder(request))
        .rejects
        .toThrow(expectedError);
    });
  });

  describe('calculateTotal', () => {
    it.each([
      { price: 100, quantity: 2, discount: 0, expected: 200 },
      { price: 100, quantity: 2, discount: 10, expected: 180 },
      { price: 99.99, quantity: 3, discount: 5, expected: 284.97 },
    ])('should calculate total: $price * $quantity - $discount% = $expected',
      ({ price, quantity, discount, expected }) => {
        const result = orderService.calculateTotal(price, quantity, discount);
        expect(result).toBeCloseTo(expected, 2);
      }
    );
  });
});

// 快照测试
describe('OrderDisplay', () => {
  it('should match snapshot', () => {
    const order = {
      id: 1,
      userId: 1,
      items: [{ name: 'Product A', price: 100 }],
      total: 100,
    };

    expect(orderService.formatOrderDisplay(order)).toMatchSnapshot();
  });
});
```

### React 组件测试

```typescript
// Button.test.tsx
import { render, screen, fireEvent } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { Button } from './Button';

describe('Button', () => {
  it('should render with text', () => {
    render(<Button>Click me</Button>);
    expect(screen.getByRole('button', { name: /click me/i })).toBeInTheDocument();
  });

  it('should call onClick when clicked', async () => {
    const handleClick = jest.fn();
    render(<Button onClick={handleClick}>Click me</Button>);

    await userEvent.click(screen.getByRole('button'));

    expect(handleClick).toHaveBeenCalledTimes(1);
  });

  it('should be disabled when disabled prop is true', () => {
    render(<Button disabled>Click me</Button>);
    expect(screen.getByRole('button')).toBeDisabled();
  });

  it('should show loading spinner when loading', () => {
    render(<Button loading>Submit</Button>);
    expect(screen.getByTestId('spinner')).toBeInTheDocument();
    expect(screen.getByRole('button')).toBeDisabled();
  });

  it('should apply variant styles', () => {
    const { rerender } = render(<Button variant="primary">Primary</Button>);
    expect(screen.getByRole('button')).toHaveClass('bg-blue-500');

    rerender(<Button variant="danger">Danger</Button>);
    expect(screen.getByRole('button')).toHaveClass('bg-red-500');
  });
});
```

### 测试覆盖率配置

```javascript
// jest.config.js
module.exports = {
  collectCoverage: true,
  coverageDirectory: 'coverage',
  coverageReporters: ['text', 'lcov', 'html'],
  coverageThreshold: {
    global: {
      branches: 80,
      functions: 80,
      lines: 80,
      statements: 80,
    },
  },
  collectCoverageFrom: [
    'src/**/*.{js,jsx,ts,tsx}',
    '!src/**/*.d.ts',
    '!src/index.ts',
    '!src/**/*.stories.{js,jsx,ts,tsx}',
  ],
};
```

## 测试原则

### AAA 模式
- **Arrange**: 准备测试数据和环境
- **Act**: 执行被测方法
- **Assert**: 验证结果

### FIRST 原则
- **Fast**: 测试执行要快
- **Independent**: 测试之间独立
- **Repeatable**: 可重复执行
- **Self-validating**: 自我验证
- **Timely**: 及时编写

## 最佳实践清单

- [ ] 每个测试只验证一个行为
- [ ] 测试方法名称清晰描述场景
- [ ] 使用 Mock 隔离外部依赖
- [ ] 覆盖正常流程和异常流程
- [ ] 覆盖边界条件
- [ ] 保持测试代码简洁
- [ ] 避免测试实现细节
- [ ] 定期运行并维护测试
