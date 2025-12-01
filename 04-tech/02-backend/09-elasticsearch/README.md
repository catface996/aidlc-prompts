# ElasticSearch 搜索引擎最佳实践

## 角色设定

你是一位精通 ElasticSearch 8.x 的搜索引擎专家，擅长索引设计、查询优化、聚合分析和集群管理。

## 提示词模板

### 索引设计

```
请帮我设计 ES 索引：
- 业务场景：[描述业务]
- 数据字段：[列出字段]
- 查询需求：[全文搜索/精确查询/范围查询/聚合]
- 数据量：[预估数据量]
- 写入频率：[实时/批量]

请提供：
1. Mapping 设计
2. 分片策略
3. 索引模板
```

### 查询优化

```
请帮我优化 ES 查询：
[粘贴查询 DSL]

当前问题：[查询慢/结果不准确/内存溢出]
索引 Mapping：[描述 Mapping]

请分析并提供优化方案。
```

## 核心代码示例

### Spring Data Elasticsearch 配置

```java
@Configuration
public class ElasticsearchConfig extends ElasticsearchConfiguration {

    @Value("${elasticsearch.uris}")
    private String[] uris;

    @Override
    public ClientConfiguration clientConfiguration() {
        return ClientConfiguration.builder()
            .connectedTo(uris)
            .withConnectTimeout(Duration.ofSeconds(5))
            .withSocketTimeout(Duration.ofSeconds(30))
            .build();
    }
}
```

### 索引定义

```java
@Document(indexName = "products")
@Setting(settingPath = "elasticsearch/product-settings.json")
@Mapping(mappingPath = "elasticsearch/product-mapping.json")
public class ProductDocument {

    @Id
    private String id;

    @Field(type = FieldType.Keyword)
    private String productId;

    @Field(type = FieldType.Text, analyzer = "ik_max_word", searchAnalyzer = "ik_smart")
    private String name;

    @Field(type = FieldType.Text, analyzer = "ik_max_word", searchAnalyzer = "ik_smart")
    private String description;

    @Field(type = FieldType.Keyword)
    private String categoryId;

    @Field(type = FieldType.Keyword)
    private String brandId;

    @Field(type = FieldType.Double)
    private BigDecimal price;

    @Field(type = FieldType.Integer)
    private Integer stock;

    @Field(type = FieldType.Integer)
    private Integer salesCount;

    @Field(type = FieldType.Keyword)
    private String status;

    @Field(type = FieldType.Nested)
    private List<ProductAttribute> attributes;

    @Field(type = FieldType.Date, format = DateFormat.date_hour_minute_second)
    private LocalDateTime createdAt;

    @Field(type = FieldType.Date, format = DateFormat.date_hour_minute_second)
    private LocalDateTime updatedAt;
}

@Data
public class ProductAttribute {
    @Field(type = FieldType.Keyword)
    private String name;

    @Field(type = FieldType.Keyword)
    private String value;
}
```

### Mapping 配置文件

```json
// resources/elasticsearch/product-mapping.json
{
  "properties": {
    "id": { "type": "keyword" },
    "productId": { "type": "keyword" },
    "name": {
      "type": "text",
      "analyzer": "ik_max_word",
      "search_analyzer": "ik_smart",
      "fields": {
        "keyword": { "type": "keyword" }
      }
    },
    "description": {
      "type": "text",
      "analyzer": "ik_max_word",
      "search_analyzer": "ik_smart"
    },
    "categoryId": { "type": "keyword" },
    "brandId": { "type": "keyword" },
    "price": { "type": "scaled_float", "scaling_factor": 100 },
    "stock": { "type": "integer" },
    "salesCount": { "type": "integer" },
    "status": { "type": "keyword" },
    "attributes": {
      "type": "nested",
      "properties": {
        "name": { "type": "keyword" },
        "value": { "type": "keyword" }
      }
    },
    "createdAt": { "type": "date", "format": "yyyy-MM-dd HH:mm:ss" },
    "updatedAt": { "type": "date", "format": "yyyy-MM-dd HH:mm:ss" }
  }
}
```

### Repository

```java
public interface ProductSearchRepository extends ElasticsearchRepository<ProductDocument, String> {

    List<ProductDocument> findByName(String name);

    Page<ProductDocument> findByCategoryId(String categoryId, Pageable pageable);

    @Query("""
        {
          "bool": {
            "must": [
              { "match": { "name": "?0" } }
            ],
            "filter": [
              { "term": { "status": "ACTIVE" } }
            ]
          }
        }
        """)
    Page<ProductDocument> searchByName(String keyword, Pageable pageable);
}
```

### 搜索服务

```java
@Service
@RequiredArgsConstructor
@Slf4j
public class ProductSearchService {

    private final ElasticsearchOperations elasticsearchOperations;

    /**
     * 复杂搜索
     */
    public SearchResult<ProductDocument> search(ProductSearchRequest request) {
        // 构建查询
        BoolQuery.Builder boolQuery = new BoolQuery.Builder();

        // 关键词搜索
        if (StringUtils.hasText(request.getKeyword())) {
            boolQuery.must(m -> m
                .multiMatch(mm -> mm
                    .query(request.getKeyword())
                    .fields("name^3", "description")
                    .type(TextQueryType.BestFields)
                )
            );
        }

        // 分类过滤
        if (StringUtils.hasText(request.getCategoryId())) {
            boolQuery.filter(f -> f
                .term(t -> t.field("categoryId").value(request.getCategoryId()))
            );
        }

        // 品牌过滤
        if (request.getBrandIds() != null && !request.getBrandIds().isEmpty()) {
            boolQuery.filter(f -> f
                .terms(t -> t
                    .field("brandId")
                    .terms(ts -> ts.value(request.getBrandIds().stream()
                        .map(FieldValue::of)
                        .toList()))
                )
            );
        }

        // 价格范围
        if (request.getMinPrice() != null || request.getMaxPrice() != null) {
            RangeQuery.Builder rangeBuilder = new RangeQuery.Builder().field("price");
            if (request.getMinPrice() != null) {
                rangeBuilder.gte(JsonData.of(request.getMinPrice()));
            }
            if (request.getMaxPrice() != null) {
                rangeBuilder.lte(JsonData.of(request.getMaxPrice()));
            }
            boolQuery.filter(f -> f.range(rangeBuilder.build()));
        }

        // 状态过滤
        boolQuery.filter(f -> f.term(t -> t.field("status").value("ACTIVE")));

        // 构建搜索请求
        NativeQuery query = NativeQuery.builder()
            .withQuery(q -> q.bool(boolQuery.build()))
            .withSort(buildSort(request.getSortBy()))
            .withPageable(PageRequest.of(request.getPage(), request.getSize()))
            .withHighlightQuery(buildHighlight())
            .build();

        SearchHits<ProductDocument> searchHits = elasticsearchOperations.search(
            query, ProductDocument.class);

        return buildSearchResult(searchHits, request);
    }

    /**
     * 聚合查询
     */
    public Map<String, List<AggregationItem>> getAggregations(String keyword) {
        NativeQuery query = NativeQuery.builder()
            .withQuery(q -> q
                .bool(b -> b
                    .must(m -> m.match(ma -> ma.field("name").query(keyword)))
                    .filter(f -> f.term(t -> t.field("status").value("ACTIVE")))
                )
            )
            .withAggregation("categories", a -> a
                .terms(t -> t.field("categoryId").size(20))
            )
            .withAggregation("brands", a -> a
                .terms(t -> t.field("brandId").size(20))
            )
            .withAggregation("price_ranges", a -> a
                .range(r -> r
                    .field("price")
                    .ranges(
                        Range.of(ra -> ra.to("100")),
                        Range.of(ra -> ra.from("100").to("500")),
                        Range.of(ra -> ra.from("500").to("1000")),
                        Range.of(ra -> ra.from("1000"))
                    )
                )
            )
            .withMaxResults(0) // 只要聚合结果
            .build();

        SearchHits<ProductDocument> searchHits = elasticsearchOperations.search(
            query, ProductDocument.class);

        return parseAggregations(searchHits.getAggregations());
    }

    /**
     * 搜索建议
     */
    public List<String> suggest(String prefix) {
        NativeQuery query = NativeQuery.builder()
            .withQuery(q -> q
                .matchPhrasePrefix(m -> m
                    .field("name")
                    .query(prefix)
                )
            )
            .withFields("name")
            .withMaxResults(10)
            .build();

        SearchHits<ProductDocument> searchHits = elasticsearchOperations.search(
            query, ProductDocument.class);

        return searchHits.getSearchHits().stream()
            .map(hit -> hit.getContent().getName())
            .distinct()
            .toList();
    }

    private Sort buildSort(String sortBy) {
        return switch (sortBy) {
            case "price_asc" -> Sort.by("price").ascending();
            case "price_desc" -> Sort.by("price").descending();
            case "sales" -> Sort.by("salesCount").descending();
            case "newest" -> Sort.by("createdAt").descending();
            default -> Sort.by("_score").descending();
        };
    }

    private HighlightQuery buildHighlight() {
        return new HighlightQuery(
            new Highlight(
                List.of(new HighlightField("name"), new HighlightField("description")),
                new HighlightParameters.HighlightParametersBuilder()
                    .withPreTags("<em>")
                    .withPostTags("</em>")
                    .build()
            ),
            ProductDocument.class
        );
    }
}
```

### 数据同步

```java
@Service
@RequiredArgsConstructor
@Slf4j
public class ProductSyncService {

    private final ProductSearchRepository searchRepository;
    private final ProductRepository productRepository;

    /**
     * 增量同步 - 监听数据库变更
     */
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void onProductChanged(ProductChangedEvent event) {
        switch (event.getType()) {
            case CREATED, UPDATED -> syncProduct(event.getProductId());
            case DELETED -> deleteProduct(event.getProductId());
        }
    }

    /**
     * 同步单个商品
     */
    public void syncProduct(Long productId) {
        Product product = productRepository.findById(productId).orElse(null);
        if (product == null) {
            deleteProduct(productId);
            return;
        }

        ProductDocument document = convertToDocument(product);
        searchRepository.save(document);
        log.info("Synced product to ES: {}", productId);
    }

    /**
     * 删除商品
     */
    public void deleteProduct(Long productId) {
        searchRepository.deleteById(productId.toString());
        log.info("Deleted product from ES: {}", productId);
    }

    /**
     * 全量同步
     */
    @Async
    public void fullSync() {
        log.info("Starting full sync...");
        int page = 0;
        int size = 1000;

        while (true) {
            Page<Product> products = productRepository.findAll(PageRequest.of(page, size));
            if (products.isEmpty()) {
                break;
            }

            List<ProductDocument> documents = products.getContent().stream()
                .map(this::convertToDocument)
                .toList();

            searchRepository.saveAll(documents);
            log.info("Synced page {}, size: {}", page, documents.size());

            page++;
        }

        log.info("Full sync completed");
    }
}
```

## 索引设计规范

| 场景 | 索引策略 | 示例 |
|------|----------|------|
| 时序数据 | 按时间滚动 | logs-2024.03.01 |
| 业务数据 | 按业务划分 | products, orders |
| 大数据量 | 按 ID 分片 | users-0, users-1 |

## 最佳实践清单

- [ ] 合理设计 Mapping，避免动态映射
- [ ] 选择合适的分词器
- [ ] 使用 keyword 类型做精确匹配
- [ ] Nested 类型用于对象数组
- [ ] 设置合理的分片数和副本数
- [ ] 使用别名管理索引
- [ ] 实现数据双写或同步机制
- [ ] 监控集群健康状态
