# Kibana 数据查询代码知识库

## 1. 基础架构设计

### 1.1 核心类设计
- 使用面向对象方式设计查询类（如 `UserDataChecker`、`KibanaDataFetcher`）
- 在类初始化时配置基础 URL 和请求头信息
- 实现主要查询方法（如 `check_user`、`fetch_data`）

### 1.2 日志配置
```python
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(levelname)s - %(message)s',
    handlers=[
        logging.FileHandler('kibana_fetch.log'),
        logging.StreamHandler()
    ]
)
logger = logging.getLogger(__name__)
```

## 2. 请求配置

### 2.1 基础 Headers 配置
```python
headers = {
    'accept': 'application/json, text/plain, */*',
    'accept-language': 'zh-CN,zh;q=0.9',
    'content-type': 'application/x-ndjson',
    'kbn-version': '6.8.23',
    'origin': 'https://kibana-tck3-next.xesv5.com',
    'referer': 'https://kibana-tck3-next.xesv5.com/app/kibana'
}
```

### 2.2 请求体构造
```python
header = {
    "index": "index-name-*",
    "ignore_unavailable": True,
    "preference": 1735210544242
}

body = {
    "version": True,
    "size": size,
    "sort": [{"@timestamp": {"order": "desc", "unmapped_type": "boolean"}}],
    "query": {
        "bool": {
            "must": [
                # 查询条件...
            ]
        }
    }
}

# NDJSON 格式发送
data = f"{json.dumps(header)}\n{json.dumps(body)}\n"
```

## 3. 数据查询技巧

### 3.1 时间范围处理
```python
# 转换时间格式为毫秒时间戳
start_ts = int(datetime.strptime(start_time, "%Y-%m-%dT%H:%M:%S.%fZ").timestamp() * 1000)
end_ts = int(datetime.strptime(end_time, "%Y-%m-%dT%H:%M:%S.%fZ").timestamp() * 1000)

# 在查询条件中使用
{
    "range": {
        "@timestamp": {
            "gte": start_ts,
            "lte": end_ts,
            "format": "epoch_millis"
        }
    }
}
```

### 3.2 分页查询实现
```python
def fetch_with_pagination(size=5000):
    page = 0
    while True:
        body = {
            "size": size,
            "from": page * size,
            # 其他查询条件...
        }
        # 发送请求获取数据
        if not hits:  # 没有更多数据时退出
            break
        page += 1
        time.sleep(0.5)  # 避免请求过快
```

### 3.3 时间分片处理
```python
time_interval = 1 * 60 * 60 * 1000  # 1小时的毫秒数
current_start = start_ts
while current_start < end_ts:
    current_end = min(current_start + time_interval, end_ts)
    # 处理当前时间分片的数据
    current_start = current_end
```

## 4. 数据处理与分析

### 4.1 使用 Pandas 处理数据
```python
# 转换数据为 DataFrame
df = pd.DataFrame(all_records)

# 基础统计分析
stats = {
    'total_records': len(df),
    'unique_users': df['uid'].nunique(),
    'success_rate': (df['response_stat'] == 1).mean() * 100,
    'avg_duration': df['x_duration'].mean()
}

# 保存数据
df.to_csv('output.csv', index=False)
```

### 4.2 JSON 数据处理
```python
try:
    param_data = json.loads(source['param'])
    response_data = json.loads(source['response'])
    # 处理数据...
except json.JSONDecodeError as e:
    logger.error(f"JSON解析错误: {str(e)}")
```

## 5. 错误处理与日志记录

### 5.1 异常处理模板
```python
try:
    # 执行查询操作
    response = requests.post(url, headers=headers, data=data)
    if response.status_code != 200:
        logger.error(f"查询失败: {response.status_code}")
        return None
    # 处理响应...
except Exception as e:
    logger.error(f"发生错误: {str(e)}", exc_info=True)
    return None
```

### 5.2 进度记录
```python
# 批量处理进度记录
processed = 0
total_users = len(user_ids)
if processed % 10 == 0:
    logger.info(f"进度: {processed}/{total_users}")
```

## 6. 最佳实践建议

1. 查询优化
   - 使用时间分片处理大量数据
   - 实现分页查询避免内存溢出
   - 添加适当的请求间隔避免频率限制

2. 错误处理
   - 实现完整的错误处理机制
   - 记录详细的错误日志
   - 添加重试机制处理临时性错误

3. 数据处理
   - 使用 Pandas 进行数据分析和统计
   - 保存中间结果避免数据丢失
   - 实现数据导出功能方便后续分析

4. 性能优化
   - 使用批量处理减少请求次数
   - 实现并发查询提高效率
   - 优化内存使用避免内存泄露

## 7. 示例用例

### 7.1 基础查询示例
```python
def basic_query(self, index_name, query_body):
    header = {"index": index_name, "ignore_unavailable": True}
    data = f"{json.dumps(header)}\n{json.dumps(query_body)}\n"
    response = requests.post(
        f"{self.base_url}",
        headers=self.headers,
        data=data
    )
    return response.json()
```

### 7.2 数据分析示例
```python
def analyze_data(df):
    stats = {
        'total_records': len(df),
        'unique_users': df['uid'].nunique(),
        'response_stats': df['response_stat'].value_counts().to_dict(),
        'avg_duration': df['x_duration'].mean()
    }
    return stats
```
