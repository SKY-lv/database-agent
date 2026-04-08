---
name: database-agent
description: "AI数据库Agent助手。SQL生成、索引优化、查询分析、数据建模。触发词：sql、数据库、查询、索引、建模、orm。"
metadata: {"openclaw": {"emoji": "🗄️"}}
---

# Database Agent

## 功能说明

AI驱动的数据库助手，处理SQL生成、优化和分析。

## 支持数据库

| 数据库 | 连接方式 |
|--------|----------|
| PostgreSQL | `postgresql://user:pass@host:5432/db` |
| MySQL | `mysql://user:pass@host:3306/db` |
| SQLite | `sqlite:///path/to/db` |
| MongoDB | `mongodb://host:27017/db` |

## 核心能力

### 1. 自然语言转SQL

```python
from sqlalchemy import create_engine, text
from openai import OpenAI

class NLToSQL:
    def __init__(self, db_url: str, model: str = "gpt-4"):
        self.engine = create_engine(db_url)
        self.client = OpenAI()
        self.schema = self._get_schema()
    
    def _get_schema(self) -> str:
        """获取数据库Schema"""
        with self.engine.connect() as conn:
            # PostgreSQL
            try:
                result = conn.execute(text("""
                    SELECT table_name, column_name, data_type, is_nullable
                    FROM information_schema.columns
                    WHERE table_schema = 'public'
                    ORDER BY table_name, ordinal_position
                """))
                return self._format_schema(result.fetchall())
            except:
                pass
            
            # MySQL
            try:
                result = conn.execute(text("SHOW TABLES"))
                tables = [r[0] for r in result.fetchall()]
                schema = ""
                for t in tables:
                    result = conn.execute(text(f"DESCRIBE {t}"))
                    schema += f"\nTable: {t}\n"
                    for row in result.fetchall():
                        schema += f"  {row[0]}: {row[1]} {'NULL' if row[3]=='YES' else 'NOT NULL'}\n"
                return schema
            except:
                pass
    
    def _format_schema(self, rows) -> str:
        tables = {}
        for row in rows:
            t, c, dt, nullable = row
            if t not in tables: tables[t] = []
            tables[t].append(f"{c} ({dt}) {'NULL' if nullable == 'YES' else 'NOT NULL'}")
        return "\n".join(f"Table: {t}\n  " + "\n  ".join(cols) for t, cols in tables.items())
    
    def query(self, nl_question: str) -> dict:
        # 获取Schema上下文
        schema_context = f"""数据库Schema:
{self.schema}

要求：
1. 只生成SELECT查询（安全）
2. 使用正确的表名和列名
3. 添加必要的WHERE条件
4. 适当使用JOIN
5. 如果需要聚合，添加GROUP BY"""
        
        response = self.client.chat.completions.create(
            model=self.model,
            messages=[
                {"role": "system", "content": "你是一个SQL专家。将自然语言转换为准确的SQL查询。"},
                {"role": "system", "content": schema_context},
                {"role": "user", "content": nl_question}
            ]
        )
        
        sql = response.choices[0].message.content
        sql = self._extract_sql(sql)
        
        # 执行查询
        with self.engine.connect() as conn:
            result = conn.execute(text(sql))
            rows = result.fetchall()
            columns = result.keys()
        
        return {
            "sql": sql,
            "columns": list(columns),
            "rows": [dict(zip(columns, row)) for row in rows],
            "count": len(rows)
        }
    
    def _extract_sql(self, text: str) -> str:
        """从回复中提取SQL"""
        import re
        match = re.search(r'```sql\s*(.*?)```', text, re.DOTALL)
        if match: return match.group(1).strip()
        match = re.search(r'SELECT.*', text, re.DOTALL | re.IGNORECASE)
        if match: return match.group(0).strip()
        return text.strip().strip('`')
```

### 2. 索引优化分析

```python
class IndexOptimizer:
    def __init__(self, engine):
        self.engine = engine
    
    def analyze_slow_queries(self) -> list:
        """分析慢查询并推荐索引"""
        with self.engine.connect() as conn:
            # PostgreSQL
            try:
                result = conn.execute(text("""
                    SELECT query, calls, mean_time, total_time
                    FROM pg_stat_statements
                    WHERE mean_time > 100
                    ORDER BY mean_time DESC
                    LIMIT 20
                """))
                queries = result.fetchall()
                
                recommendations = []
                for q in queries:
                    analysis = self._analyze_query(q[0])
                    if analysis['recommendation']:
                        recommendations.append(analysis)
                return recommendations
            except: pass
        return []
    
    def _analyze_query(self, sql: str) -> dict:
        """分析单条查询"""
        import re
        
        # 提取WHERE条件和JOIN条件
        where_cols = re.findall(r'WHERE\s+(\w+)\.(\w+)\s*=', sql, re.I)
        join_cols = re.findall(r'JOIN\s+(\w+)\s+ON\s+\w+\.(\w+)', sql, re.I)
        order_cols = re.findall(r'ORDER BY\s+(\w+)', sql, re.I)
        
        suggestions = []
        for table, col in where_cols:
            suggestions.append(f"CREATE INDEX idx_{table}_{col} ON {table}({col});")
        
        return {
            "sql": sql,
            "where_conditions": where_cols,
            "join_conditions": join_cols,
            "order_columns": order_cols,
            "recommendation": suggestions
        }
    
    def generate_create_index(self, table: str, columns: list, unique: bool = False) -> str:
        cols = ", ".join(columns)
        idx_name = f"idx_{table}_{'_'.join(columns)}"
        uniq = "UNIQUE " if unique else ""
        return f"CREATE {uniq}INDEX {idx_name} ON {table}({cols});"
```

### 3. 数据建模

```python
class DataModeler:
    def __init__(self, llm):
        self.llm = llm
    
    def design_from_description(self, description: str) -> dict:
        """从业务描述设计数据模型"""
        prompt = f"""根据以下业务描述，设计关系型数据库模型：

业务描述：
{description}

要求：
1. 识别实体（Entity）
2. 确定实体的属性（Attribute）
3. 识别关系（Relationship）
4. 确定主键和外键
5. 选择合适的数据类型

输出JSON格式：
{{
  "entities": [
    {{
      "name": "实体名(英文)",
      "columns": [
        {{"name": "列名", "type": "类型", "pk": true, "fk": null, "nullable": false, "unique": false}}
      ],
      "relations": [
        {{"to": "关联实体", "type": "one_to_many|many_to_many|one_to_one", "via": "中间表(如有)"}}
      ]
    }}
  ]
}}"""
        
        response = self.llm.chat.completions.create(
            model="gpt-4",
            messages=[{"role": "user", "content": prompt}]
        )
        
        import json
        model = json.loads(response.choices[0].message.content)
        return model
    
    def generate_migrations(self, model: dict) -> list[str]:
        """生成数据库迁移SQL"""
        migrations = []
        
        for entity in model.get('entities', []):
            table_name = entity['name'].lower()
            
            # CREATE TABLE
            cols = []
            pks = []
            fks = []
            
            for col in entity.get('columns', []):
                col_def = f"{col['name']} {col['type']}"
                if not col.get('nullable', True): col_def += " NOT NULL"
                if col.get('unique'): col_def += " UNIQUE"
                if col.get('pk'): pks.append(col['name'])
                cols.append(col_def)
            
            # Primary Key
            if pks:
                cols.append(f"PRIMARY KEY ({', '.join(pks)})")
            
            # Foreign Keys
            for rel in entity.get('relations', []):
                ref_table = rel['to'].lower()
                cols.append(f"FOREIGN KEY ({rel.get('local_key', 'id')}) REFERENCES {ref_table}(id)")
            
            sql = f"CREATE TABLE IF NOT EXISTS {table_name} (\n  " + ",\n  ".join(cols) + "\n);"
            migrations.append(sql)
            
            # Relation tables (many-to-many)
            if rel.get('type') == 'many_to_many':
                mid_table = f"{table_name}_{ref_table}"
                mid_sql = f"CREATE TABLE IF NOT EXISTS {mid_table} (\n  {table_name}_id INT NOT NULL,\n  {ref_table}_id INT NOT NULL,\n  PRIMARY KEY ({table_name}_id, {ref_table}_id)\n);"
                migrations.append(mid_sql)
        
        return migrations
```

### 4. 查询解释分析

```python
def explain_query(engine, sql: str) -> dict:
    """执行EXPLAIN分析查询计划"""
    with engine.connect() as conn:
        explain_sql = f"EXPLAIN (FORMAT JSON) {sql}"
        try:
            result = conn.execute(text(explain_sql))
            plan = result.fetchone()[0]
            
            # 分析成本
            total_cost = plan[0]['Plan']['Total Cost']
            rows_estimated = plan[0]['Plan']['Plan Rows']
            
            return {
                "plan": plan,
                "estimated_cost": total_cost,
                "estimated_rows": rows_estimated,
                "warnings": [],
                "recommendations": []
            }
        except:
            pass
        
        # MySQL fallback
        result = conn.execute(text(f"EXPLAIN {sql}"))
        return {"plan": [dict(row._mapping) for row in result.fetchall()]}
```

## 常见任务模板

### 数据统计
```sql
-- 用户活跃统计
SELECT 
    DATE(created_at) as date,
    COUNT(*) as new_users,
    SUM(is_active::int) as active_users
FROM users
WHERE created_at >= CURRENT_DATE - INTERVAL '30 days'
GROUP BY DATE(created_at)
ORDER BY date;
```

### 数据清洗
```sql
-- 查找并删除重复记录
WITH duplicates AS (
    SELECT id, ROW_NUMBER() OVER (PARTITION BY email ORDER BY created_at) as rn
    FROM users
)
DELETE FROM users WHERE id IN (
    SELECT id FROM duplicates WHERE rn > 1
);
```

### 性能诊断
```sql
-- 查找缺失索引的外键
SELECT 
    tc.table_name, kcu.column_name,
    ccu.table_name AS foreign_table_name,
    ccu.column_name AS foreign_column_name
FROM information_schema.table_constraints AS tc
JOIN information_schema.key_column_usage AS kcu ON tc.constraint_name = kcu.constraint_name
JOIN information_schema.constraint_column_usage AS ccu ON ccu.constraint_name = tc.constraint_name
WHERE constraint_type = 'FOREIGN KEY'
AND tc.table_schema = 'public';
```
