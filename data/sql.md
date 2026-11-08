---
date created: 2026-10-25T14:34
date updated: 2026-11-06T18:18
---

# SQL 基础与查询

> 用途：用统一的轻量示例练习常见 SQL 查询、事务、表结构和性能基础。
> 适用类型：知识 / 实践
> 更新：2026-10-25

不同数据库在分页、日期、字符串函数和部分 DDL 上存在差异；示例优先使用常见 SQL，遇到方言差异会单独注明。

## 学习优先级

| 优先级 | 内容 | 建议时间 | 掌握标准 |
| --- | --- | ---: | --- |
| P0 | `SELECT`、`WHERE`、`ORDER BY`、`LIMIT`、`NULL` | 1.5 小时 | 能独立完成筛选、排序和分页 |
| P0 | 聚合函数、`GROUP BY`、`HAVING` | 1.5 小时 | 能完成分类统计和聚合条件过滤 |
| P0 | `INNER JOIN`、`LEFT JOIN` | 2 小时 | 能根据表关系写出多表查询 |
| P1 | `CASE`、子查询、`EXISTS`、CTE | 2 小时 | 能拆解稍复杂的业务查询 |
| P1 | 窗口函数 | 2 小时 | 能完成排名、组内 Top N、累计值 |
| P1 | 增删改、事务、索引、`EXPLAIN` | 1.5 小时 | 理解数据修改和基本性能分析 |
| P2 | 去重、集合运算、日期与字符串函数 | 1 小时 | 能处理常见数据清洗题 |

### 推荐安排

1. **第一天** 完成全部 P0 内容，每个例子手写一遍。
2. **第二天** 学习 `CASE`、子查询、CTE 和窗口函数。
3. **第三天** 补充事务、索引，并限时完成 10～15 道 SQL 题。
4. **面试前** 重点复习连接、分组、窗口函数和 `NULL` 陷阱。

---

## 统一示例数据

```sql
CREATE TABLE departments (
    dept_id INTEGER PRIMARY KEY,
    name VARCHAR(50) NOT NULL UNIQUE
);

CREATE TABLE employees (
    id INTEGER PRIMARY KEY,
    name VARCHAR(50) NOT NULL,
    dept_id INTEGER,
    salary DECIMAL(10, 2) NOT NULL CHECK (salary >= 0),
    hire_date DATE NOT NULL,
    manager_id INTEGER,
    FOREIGN KEY (dept_id) REFERENCES departments(dept_id),
    FOREIGN KEY (manager_id) REFERENCES employees(id)
);

CREATE TABLE test_tasks (
    id INTEGER PRIMARY KEY,
    employee_id INTEGER NOT NULL,
    project VARCHAR(50) NOT NULL,
    status VARCHAR(20) NOT NULL
        CHECK (status IN ('passed', 'failed', 'running')),
    defects INTEGER NOT NULL DEFAULT 0 CHECK (defects >= 0),
    finished_at DATE,
    FOREIGN KEY (employee_id) REFERENCES employees(id)
);

INSERT INTO departments (dept_id, name) VALUES
    (1, '测试部'), (2, '开发部'), (3, '产品部'), (4, '运维部');

INSERT INTO employees
    (id, name, dept_id, salary, hire_date, manager_id)
VALUES
    (1, '张三', 1, 12000, '2024-01-10', 3),
    (2, '李四', 1, 15000, '2023-06-01', 3),
    (3, '王五', 1, 22000, '2022-03-15', NULL),
    (4, '赵六', 2, 18000, '2023-11-20', 6),
    (5, '陈七', 2, 18000, '2024-04-08', 6),
    (6, '周八', 2, 26000, '2021-09-01', NULL),
    (7, '吴九', 3, 14000, '2024-02-18', NULL),
    (8, '郑十', NULL, 10000, '2025-01-06', NULL);

INSERT INTO test_tasks
    (id, employee_id, project, status, defects, finished_at)
VALUES
    (101, 1, 'Alpha', 'passed', 3, '2025-05-01'),
    (102, 1, 'Beta', 'failed', 5, '2025-05-03'),
    (103, 2, 'Alpha', 'passed', 2, '2025-05-04'),
    (104, 2, 'Gamma', 'passed', 4, '2025-05-06'),
    (105, 3, 'Beta', 'passed', 1, '2025-05-07'),
    (106, 4, 'Alpha', 'running', 0, NULL),
    (107, 4, 'Gamma', 'failed', 6, '2025-05-09'),
    (108, 5, 'Beta', 'passed', 2, '2025-05-10');

```

后续例子共用以下三张表，避免每节重复定义数据：

- `departments → employees`：一对多，可练习部门统计和外连接。
- `employees → employees`：通过 `manager_id` 自关联，可练习层级关系。
- `employees → test_tasks`：一对多，可练习任务统计、状态筛选和窗口函数。

### departments：部门表

| dept_id | name |
| ---: | --- |
| 1 | 测试部 |
| 2 | 开发部 |
| 3 | 产品部 |
| 4 | 运维部 |

### employees：员工表

| id | name | dept_id | salary | hire_date | manager_id |
| ---: | --- | ---: | ---: | --- | ---: |
| 1 | 张三 | 1 | 12000 | 2024-01-10 | 3 |
| 2 | 李四 | 1 | 15000 | 2023-06-01 | 3 |
| 3 | 王五 | 1 | 22000 | 2022-03-15 | `NULL` |
| 4 | 赵六 | 2 | 18000 | 2023-11-20 | 6 |
| 5 | 陈七 | 2 | 18000 | 2024-04-08 | 6 |
| 6 | 周八 | 2 | 26000 | 2021-09-01 | `NULL` |
| 7 | 吴九 | 3 | 14000 | 2024-02-18 | `NULL` |
| 8 | 郑十 | `NULL` | 10000 | 2025-01-06 | `NULL` |

### test_tasks：测试任务表

| id | employee_id | project | status | defects | finished_at |
| ---: | ---: | --- | --- | ---: | --- |
| 101 | 1 | Alpha | passed | 3 | 2025-05-01 |
| 102 | 1 | Beta | failed | 5 | 2025-05-03 |
| 103 | 2 | Alpha | passed | 2 | 2025-05-04 |
| 104 | 2 | Gamma | passed | 4 | 2025-05-06 |
| 105 | 3 | Beta | passed | 1 | 2025-05-07 |
| 106 | 4 | Alpha | running | 0 | `NULL` |
| 107 | 4 | Gamma | failed | 6 | 2025-05-09 |
| 108 | 5 | Beta | passed | 2 | 2025-05-10 |

---

## P0：基础查询

### SELECT：选择列

- 从表中读取数据。
- 尽量明确列名；生产代码不建议长期使用 `SELECT *`。
- `AS` 为列或表设置别名。

```sql
SELECT name, salary
FROM employees;

SELECT name AS employee_name, salary * 12 AS annual_salary
FROM employees;
```

### DISTINCT：结果去重

- 去除查询结果中的重复组合。
- 多列 `DISTINCT` 会对多列组合整体去重。

```sql
SELECT DISTINCT status
FROM test_tasks;

SELECT DISTINCT project, status
FROM test_tasks;
```

### WHERE：行过滤

- 在分组和聚合之前筛选原始数据。
- 常配合比较运算符、`AND`、`OR`、`IN`、`BETWEEN` 和 `LIKE`。

```sql
SELECT name, salary
FROM employees
WHERE salary >= 15000
  AND dept_id IN (1, 2);
```

```sql
SELECT name, hire_date
FROM employees
WHERE hire_date BETWEEN '2023-01-01' AND '2024-12-31';
```

```sql
-- % 表示任意长度字符，_ 表示一个字符
SELECT name
FROM employees
WHERE name LIKE '张%';
```

> `AND` 的优先级高于 `OR`。条件复杂时主动加括号，不依赖记忆优先级。

```sql
SELECT *
FROM test_tasks
WHERE project = 'Alpha'
  AND (status = 'passed' OR status = 'running');
```

### NULL：缺失值判断

- `NULL` 表示未知或缺失，不等于空字符串，也不等于数字 `0`。
- 必须使用 `IS NULL` 或 `IS NOT NULL`，不能使用等于号。
- 普通计算或比较遇到 `NULL`，结果通常仍是未知。

```sql
SELECT name
FROM employees
WHERE dept_id IS NULL;
```

```sql
-- COALESCE 返回第一个非 NULL 值
SELECT name, COALESCE(dept_id, 0) AS dept_id
FROM employees;
```

### ORDER BY：结果排序

- `ASC` 升序，默认值；`DESC` 降序。
- 可以按多列排序，前一列相同时再比较后一列。

```sql
SELECT name, salary
FROM employees
ORDER BY salary DESC, id ASC;
```

### LIMIT：限制结果数量

- 常用于 Top N 和分页。
- `LIMIT` 是常见分页写法；其他数据库可能使用 `TOP` 或 `OFFSET … FETCH`。

```sql
-- 工资最高的 3 名员工
SELECT name, salary
FROM employees
ORDER BY salary DESC
LIMIT 3;
```

```sql
-- 每页 3 条，读取第 2 页
SELECT id, name
FROM employees
ORDER BY id
LIMIT 3 OFFSET 3;
```

> 大数据量下，较深的 `OFFSET` 可能很慢。生产系统常根据上一页最后一个 ID 做游标分页。

例如上一页最后一个 ID 是 `10000`，下一页直接从它之后取值：

```sql
SELECT id, name
FROM employees
WHERE id > 10000
ORDER BY id
LIMIT 20;
```

按其他字段排序时，游标必须带齐排序键及唯一的平局规则，不能只记录 ID。

---

## P0：聚合与分组

### 聚合函数

| 函数            | 作用           | 注意点                   |
|-----------------|----------------|--------------------------|
| `COUNT(*)`      | 统计行数       | 包含各列为 `NULL` 的行   |
| `COUNT(column)` | 统计非空值数量 | 不统计该列为 `NULL` 的行 |
| `SUM(column)`   | 求和           | 常用于金额、数量         |
| `AVG(column)`   | 平均值         | 忽略 `NULL`              |
| `MIN(column)`   | 最小值         | 可用于数字、日期等       |
| `MAX(column)`   | 最大值         | 可用于数字、日期等       |

```sql
SELECT
    COUNT(*) AS employee_count,
    COUNT(dept_id) AS assigned_count,
    AVG(salary) AS average_salary,
    MAX(salary) AS highest_salary
FROM employees;
```

### GROUP BY：分类统计

- 将相同分组键的数据放在一组，再分别聚合。
- 常用于“每个部门”“每个项目”“每天”的统计。
- `SELECT` 中未聚合的普通列，通常必须出现在 `GROUP BY` 中。

```sql
-- 每个部门的人数和平均工资
SELECT
    dept_id,
    COUNT(*) AS employee_count,
    AVG(salary) AS average_salary
FROM employees
GROUP BY dept_id;
```

```sql
-- 每个项目中各状态的任务数量
SELECT project, status, COUNT(*) AS task_count
FROM test_tasks
GROUP BY project, status
ORDER BY project, status;
```

### HAVING：过滤分组结果

- `WHERE` 过滤原始行；`HAVING` 过滤聚合后的组。
- 如果条件不依赖聚合结果，优先放在 `WHERE`，减少参与聚合的数据量。

```sql
-- 平均工资不低于 18000 的部门
SELECT dept_id, AVG(salary) AS average_salary
FROM employees
GROUP BY dept_id
HAVING AVG(salary) >= 18000;
```

```sql
-- 仅统计已完成任务，并找出完成任务数不少于 2 的员工
SELECT employee_id, COUNT(*) AS finished_count
FROM test_tasks
WHERE status IN ('passed', 'failed')
GROUP BY employee_id
HAVING COUNT(*) >= 2;
```

### SQL 书写顺序与执行逻辑

书写顺序：

```text
SELECT → FROM → JOIN → WHERE → GROUP BY → HAVING → ORDER BY → LIMIT
```

便于理解的逻辑执行顺序：

```text
FROM / JOIN → WHERE → GROUP BY → HAVING → SELECT → ORDER BY → LIMIT
```

这解释了为什么多数数据库不能在同层 `WHERE` 中直接使用 `SELECT` 别名。

---

## P0：多表连接

### INNER JOIN：只保留匹配数据

- 返回两张表中连接条件匹配的行。
- 适用于“只关心有部门的员工”“只关心执行过任务的员工”。

```sql
SELECT e.name, d.name AS department_name
FROM employees AS e
INNER JOIN departments AS d
    ON e.dept_id = d.dept_id;
```

结果不包含没有部门的郑十，也不包含没有员工的运维部。

### LEFT JOIN：保留左表全部数据

- 左表数据全部保留，右表没有匹配时对应列为 `NULL`。
- 适用于查找“没有关联记录”的数据。

```sql
-- 展示所有员工，即使员工没有部门
SELECT e.name, d.name AS department_name
FROM employees AS e
LEFT JOIN departments AS d
    ON e.dept_id = d.dept_id;
```

```sql
-- 找出没有员工的部门
SELECT d.dept_id, d.name
FROM departments AS d
LEFT JOIN employees AS e
    ON e.dept_id = d.dept_id
WHERE e.id IS NULL;
```

### RIGHT JOIN：保留右表全部数据

- 与 `LEFT JOIN` 方向相反：右表全部保留，左表没有匹配时对应列为 `NULL`。
- 大多数场景可以交换表的位置，改写为更常见的 `LEFT JOIN`。

```sql
-- 保留所有部门，包括没有员工的运维部
SELECT
    e.name AS employee_name,
    d.name AS department_name
FROM employees AS e
RIGHT JOIN departments AS d
    ON e.dept_id = d.dept_id;
```

等价的 `LEFT JOIN` 写法通常更易读：

```sql
SELECT
    e.name AS employee_name,
    d.name AS department_name
FROM departments AS d
LEFT JOIN employees AS e
    ON e.dept_id = d.dept_id;
```

### FULL OUTER JOIN：保留两侧全部数据

- 返回两表匹配的数据，以及左右两侧未匹配的数据。
- 适合数据核对、差异比较和寻找两侧缺失记录。

```sql
SELECT
    e.name AS employee_name,
    d.name AS department_name
FROM employees AS e
FULL OUTER JOIN departments AS d
    ON e.dept_id = d.dept_id;
```

在示例数据中：

- 郑十没有部门，因此 `department_name` 为 `NULL`。
- 运维部没有员工，因此 `employee_name` 为 `NULL`。

不支持 `FULL OUTER JOIN` 的数据库可以组合两个外连接：

```sql
SELECT e.name AS employee_name, d.name AS department_name
FROM employees AS e
LEFT JOIN departments AS d
    ON e.dept_id = d.dept_id

UNION ALL

SELECT e.name AS employee_name, d.name AS department_name
FROM departments AS d
LEFT JOIN employees AS e
    ON e.dept_id = d.dept_id
WHERE e.id IS NULL;
```

> 不同数据库对 `RIGHT JOIN` 和 `FULL OUTER JOIN` 的支持不同；不支持时可以使用等价的 `LEFT JOIN` 或 `UNION ALL` 改写。

### ON 与 WHERE 的位置陷阱

如果希望保留没有通过任务的员工，右表过滤条件应放在 `ON` 中：

```sql
SELECT e.name, t.id AS passed_task_id
FROM employees AS e
LEFT JOIN test_tasks AS t
    ON t.employee_id = e.id
   AND t.status = 'passed';
```

如果把 `t.status = 'passed'` 放进 `WHERE`，右表为空的行也会被过滤，效果接近 `INNER JOIN`。

### SELF JOIN：同一张表连接自己

- 常用于员工与经理、分类与父分类等层级关系。

```sql
SELECT
    e.name AS employee_name,
    m.name AS manager_name
FROM employees AS e
LEFT JOIN employees AS m
    ON e.manager_id = m.id;
```

### CROSS JOIN：笛卡尔积

- 左表每一行与右表每一行组合。
- 若左表有 3 行、右表有 4 行，结果为 12 行。
- 常用于生成“日期 × 产品”“环境 × 用例”等全部组合；误漏连接条件也可能意外产生笛卡尔积。

```sql
SELECT e.name, d.name AS department_name
FROM employees AS e
CROSS JOIN departments AS d;
```

### USING：同名连接列的简写

两张表的连接列名称相同时，可以用 `USING` 简化 `ON`：

```sql
SELECT *
FROM employees
JOIN departments USING (dept_id);
```

等价于：

```sql
SELECT *
FROM employees AS e
JOIN departments AS d
    ON e.dept_id = d.dept_id;
```

> 实际项目中 `ON` 更明确，也支持不同列名和多个复杂条件，因此更通用。不建议依赖会自动推断同名列的 `NATURAL JOIN`。

### 一对多连接后的重复

一个员工对应多个任务，连接后员工会出现多行，这是正常的一对多展开，不应盲目使用 `DISTINCT` 掩盖。

```sql
SELECT e.name, COUNT(t.id) AS task_count
FROM employees AS e
LEFT JOIN test_tasks AS t
    ON t.employee_id = e.id
GROUP BY e.id, e.name;
```

---

## P1：条件表达式与复杂查询

### CASE：条件分类

- 类似 Python 的 `if / elif / else`。
- 常用于分档、条件统计和结果转换。

```sql
SELECT
    name,
    salary,
    CASE
        WHEN salary >= 20000 THEN '高'
        WHEN salary >= 15000 THEN '中'
        ELSE '低'
    END AS salary_level
FROM employees;
```

#### 条件聚合

```sql
SELECT
    project,
    COUNT(*) AS total_count,
    SUM(CASE WHEN status = 'passed' THEN 1 ELSE 0 END) AS passed_count,
    SUM(CASE WHEN status = 'failed' THEN 1 ELSE 0 END) AS failed_count
FROM test_tasks
GROUP BY project;
```

### 子查询：把查询结果交给另一层查询

#### 标量子查询

返回单个值。

```sql
-- 工资高于全公司平均工资的员工
SELECT name, salary
FROM employees
WHERE salary > (
    SELECT AVG(salary)
    FROM employees
);
```

#### IN 子查询

```sql
-- 执行过失败任务的员工
SELECT name
FROM employees
WHERE id IN (
    SELECT employee_id
    FROM test_tasks
    WHERE status = 'failed'
);
```

#### 非相关子查询与相关子查询

判断标准是：**子查询是否引用外层查询的字段**。

| 类型         | 是否引用外层字段 | 便于理解的逻辑顺序                               |
|--------------|------------------|--------------------------------------------------|
| 非相关子查询 | 否               | 内层先得到结果，外层再使用结果                   |
| 相关子查询   | 是               | 外层取一行，内层根据当前行判断，外层决定是否保留 |

上面的 `IN` 示例属于非相关子查询。内层可以独立得到失败任务的员工编号，再交给外层筛选：

```text
内层查询得到 (1, 4) → 外层查询 id IN (1, 4)
```

下面的 `EXISTS` 示例属于相关子查询，因为内层引用了外层当前员工的 `e.id`：

```sql
t.employee_id = e.id
```

逻辑上可以理解为：

```text
外层取一名员工
→ 内层使用该员工的 e.id 查找失败任务
→ 至少找到一行时 EXISTS 为 TRUE，否则为 FALSE
→ 外层保留或过滤该员工
```

这是便于理解的逻辑模型。数据库优化器可能把 `IN` 或 `EXISTS` 改写成连接或半连接，实际物理执行顺序应以 `EXPLAIN` 为准。

### EXISTS：判断关联记录是否存在

- `EXISTS (子查询)` 本身就是布尔条件：子查询至少返回一行则为 `TRUE`，没有返回行则为 `FALSE`。
- 找到第一条匹配记录即可判断成立。
- 经常用于“存在”“从未存在”等问题。

```sql
SELECT e.name
FROM employees AS e
WHERE EXISTS (
    SELECT 1
    FROM test_tasks AS t
    WHERE t.employee_id = e.id
      AND t.status = 'failed'
);
```

其中 `SELECT 1` 的值没有特殊意义，`EXISTS` 只检查有没有返回行；真正关联当前员工的是 `t.employee_id = e.id`。

```sql
-- 从未执行过任务的员工
SELECT e.name
FROM employees AS e
WHERE NOT EXISTS (
    SELECT 1
    FROM test_tasks AS t
    WHERE t.employee_id = e.id
);
```

> 当子查询可能包含 `NULL` 时，`NOT EXISTS` 通常比 `NOT IN` 更不容易踩坑。

### CTE：为复杂查询分步骤命名

- 使用 `WITH` 定义临时结果集，仅在当前语句中有效。
- 比多层嵌套子查询更易读，也便于逐段调试。

```sql
WITH employee_task_stats AS (
    SELECT
        employee_id,
        COUNT(*) AS task_count,
        SUM(defects) AS total_defects
    FROM test_tasks
    GROUP BY employee_id
)
SELECT e.name, s.task_count, s.total_defects
FROM employee_task_stats AS s
JOIN employees AS e
    ON e.id = s.employee_id
WHERE s.task_count >= 2;
```

### 窗口函数、子查询与 CTE：如何选择

先问一个问题：**最终结果是否需要保留明细行？**

| 需求                             | 推荐写法                | 结果形态        |
|----------------------------------|-------------------------|-----------------|
| 分组后只要汇总结果               | `GROUP BY`              | 每组通常一行    |
| 保留明细，同时附带组内统计       | 窗口函数                | 明细行仍保留    |
| 组内排名、每组 Top N、累计值     | 窗口函数                | 明细行 + 计算列 |
| 判断是否存在关联记录             | `EXISTS` / `NOT EXISTS` | 过滤外层行      |
| 先汇总，再与明细连接             | 子查询或 CTE + `JOIN`   | 中间结果 + 明细 |
| 查询步骤较多，需要给中间结果命名 | CTE                     | 结构更清晰      |

例如，部门工资总额只需要一行时使用 `GROUP BY`：

```sql
SELECT dept_id, SUM(salary) AS department_salary
FROM employees
GROUP BY dept_id;
```

如果每名员工都要保留，同时显示部门工资总额，使用窗口函数：

```sql
SELECT
    name,
    dept_id,
    salary,
    SUM(salary) OVER (
        PARTITION BY dept_id
    ) AS department_salary,
    salary * 1.0 / SUM(salary) OVER (
        PARTITION BY dept_id
    ) AS salary_ratio
FROM employees;
```

不用窗口函数时，可以先汇总，再连接回员工明细：

```sql
WITH department_stats AS (
    SELECT dept_id, SUM(salary) AS department_salary
    FROM employees
    GROUP BY dept_id
)
SELECT
    e.name,
    e.dept_id,
    e.salary,
    s.department_salary,
    e.salary * 1.0 / s.department_salary AS salary_ratio
FROM employees AS e
JOIN department_stats AS s
    ON e.dept_id = s.dept_id;
```

这三种写法的区别可以记成：

```text
GROUP BY：分组后压缩
窗口函数：分组计算，但保留明细
子查询 / CTE：把一个查询结果交给另一层使用
```

窗口函数和 CTE 并不冲突，常见组合是：CTE 先计算排名，外层再过滤排名结果。

---

## P1：窗口函数

窗口函数不会像 `GROUP BY` 一样把多行压缩成一行，而是在保留原始行的同时计算排名、累计值或组内统计。

基本结构：

```sql
函数() OVER (
    PARTITION BY 分组列
    ORDER BY 排序列
)
```

### 哪些函数可以不加 `OVER`

#### 聚合函数：可以加，也可以不加

`COUNT`、`SUM`、`AVG`、`MAX`、`MIN` 本身就是普通聚合函数：

```sql
-- 不加 OVER：聚合并压缩结果
SELECT dept_id, SUM(salary) AS department_salary
FROM employees
GROUP BY dept_id;
```

加上 `OVER` 后，它们变成窗口聚合函数：

```sql
-- 加 OVER：保留员工明细，同时附加部门总工资
SELECT
    name,
    dept_id,
    salary,
    SUM(salary) OVER (
        PARTITION BY dept_id
    ) AS department_salary
FROM employees;
```

可以这样记：

```text
SUM(salary)                    → 分组汇总，行数通常减少
SUM(salary) OVER (...)         → 窗口计算，保留原始行
```

#### 专用窗口函数：必须加 `OVER`

`ROW_NUMBER`、`RANK`、`DENSE_RANK`、`LAG`、`LEAD` 等函数本身依赖窗口范围或行顺序，必须写 `OVER`：

```sql
SELECT
    name,
    salary,
    LAG(salary) OVER (
        ORDER BY hire_date
    ) AS previous_salary
FROM employees;
```

不能写成：

```sql
-- 错误：LAG 必须配合 OVER
SELECT LAG(salary)
FROM employees;
```

原因是数据库必须知道“上一行”或“下一行”按照什么顺序确定；这个信息由 `OVER (ORDER BY …)` 提供。

### 常见使用场景

- `ROW_NUMBER()`：每组只取固定数量的行，例如每个部门工资最高的两名员工。
- `RANK()`：需要保留并列名次，并接受并列后跳号，例如竞赛排名。
- `DENSE_RANK()`：需要保留并列名次，但不希望跳号，例如每个部门的最高工资档位。
- `SUM / AVG / COUNT … OVER`：保留明细的同时计算组内总和、平均值或数量。
- `LAG / LEAD`：比较当前行与上一行、下一行，例如相邻任务的缺陷数变化。

如果只是得到每个部门一行的统计结果，不需要窗口函数，直接使用 `GROUP BY`；如果需要根据窗口计算结果进行筛选，通常先放进 CTE 或派生表，再在外层使用 `WHERE`。

### ROW_NUMBER、RANK、DENSE_RANK

| 函数           | 相同值处理       | 示例排名 |
|----------------|------------------|----------|
| `ROW_NUMBER()` | 仍分配不同序号   | 1、2、3  |
| `RANK()`       | 并列，后续跳号   | 1、1、3  |
| `DENSE_RANK()` | 并列，后续不跳号 | 1、1、2  |

```sql
SELECT
    name,
    dept_id,
    salary,
    DENSE_RANK() OVER (
        PARTITION BY dept_id
        ORDER BY salary DESC
    ) AS salary_rank
FROM employees;
```

### 组内 Top N

```sql
WITH ranked_employees AS (
    SELECT
        id,
        name,
        dept_id,
        salary,
        ROW_NUMBER() OVER (
            PARTITION BY dept_id
            ORDER BY salary DESC, id ASC
        ) AS row_num
    FROM employees
    WHERE dept_id IS NOT NULL
)
SELECT name, dept_id, salary
FROM ranked_employees
WHERE row_num <= 2;
```

### 分组总数与占比

```sql
SELECT
    name,
    dept_id,
    salary,
    SUM(salary) OVER (PARTITION BY dept_id) AS department_salary,
    salary * 1.0
        / SUM(salary) OVER (PARTITION BY dept_id) AS salary_ratio
FROM employees;
```

### LAG / LEAD：读取前后行

- 常用于环比、前后版本对比、连续登录或相邻记录分析。

```sql
SELECT
    employee_id,
    finished_at,
    defects,
    LAG(defects) OVER (
        PARTITION BY employee_id
        ORDER BY finished_at
    ) AS previous_defects
FROM test_tasks
WHERE finished_at IS NOT NULL;
```

---

## P1：增删改与事务

### INSERT：新增数据

```sql
INSERT INTO departments (dept_id, name)
VALUES (5, '质量部');
```

### UPDATE：更新数据

```sql
UPDATE employees
SET salary = salary + 1000
WHERE dept_id = 1;
```

同时修改多个字段：

```sql
UPDATE test_tasks
SET status = 'passed',
    finished_at = '2025-05-12'
WHERE id = 106;
```

### DELETE：删除数据

```sql
DELETE FROM test_tasks
WHERE status = 'running'
  AND finished_at IS NULL;
```

> 执行 `UPDATE` 或 `DELETE` 前，先用相同的 `WHERE` 写一条 `SELECT`，确认影响范围。

### 事务：让多步操作整体成功或失败

事务可以分阶段执行，但必须使用同一个数据库连接：先开启事务并执行修改，再检查结果，最后根据检查结果二选一执行 `COMMIT` 或 `ROLLBACK`。

第一阶段：执行修改并检查，**先不要提交**：

```sql
BEGIN;

UPDATE employees
SET salary = salary + 500
WHERE dept_id = 1;

SELECT *
FROM employees
WHERE dept_id = 1;
```

第二阶段：根据检查结果，手动选择一个：

```sql
-- 确认结果正确：提交修改
COMMIT;

-- 发现结果错误：撤销本次事务
ROLLBACK;
```

`COMMIT` 和 `ROLLBACK` 是二选一，不是两条都执行。只写注释不会触发回滚；只有真正执行 `ROLLBACK` 语句才会撤销事务。

#### ACID 简记

| 特性               | 含义                               |
|--------------------|------------------------------------|
| 原子性 Atomicity   | 一组操作要么全部成功，要么全部回滚 |
| 一致性 Consistency | 事务前后数据约束保持成立           |
| 隔离性 Isolation   | 并发事务之间尽量互不干扰           |
| 持久性 Durability  | 已提交的数据不会因普通故障丢失     |

面试中还应知道：脏读、不可重复读、幻读，以及数据库通过隔离级别在一致性与并发性能之间取舍。

#### 并发读取问题

假设 `T1` 和 `T2` 是两个同时执行的事务。

##### 脏读：读到了未提交的数据

| 时刻 | `T1`         | `T2`                            |
|------|--------------|---------------------------------|
| 1    |              | `BEGIN`，把员工工资改成 `20000` |
| 2    | 读到 `20000` |                                 |
| 3    |              | `ROLLBACK`，工资恢复原值        |

`T1` 读到的 `20000` 从未真正提交，因此是脏数据。

##### 不可重复读：同一行的值变了

```sql
SELECT salary
FROM employees
WHERE id = 1;
```

| 时刻 | `T1`                         | `T2`                       |
|------|------------------------------|----------------------------|
| 1    | `BEGIN`，第一次读取：`12000` |                            |
| 2    |                              | 修改为 `15000` 并 `COMMIT` |
| 3    | 第二次读取：`15000`          |                            |

两次查询针对的是**同一行**，但这一行的值发生了变化，所以叫不可重复读。

##### 幻读：同一范围内的行变了

```sql
SELECT id, salary
FROM employees
WHERE salary >= 18000;
```

| 时刻 | `T1`                         | `T2`                                   |
|------|------------------------------|----------------------------------------|
| 1    | `BEGIN`，第一次查询得到 2 行 |                                        |
| 2    |                              | 插入一名工资 `20000` 的员工并 `COMMIT` |
| 3    | 第二次查询得到 3 行          |                                        |

两次查询条件相同，但符合条件的**行集合**发生了变化，就像多出了一行“幻影”，所以叫幻读。删除匹配行也可能造成幻读。

#### 不可重复读和幻读的区别

```text
不可重复读：关注同一行的字段值是否变化
幻读：关注同一查询范围内，行的出现或消失
```

| 对比     | 不可重复读                           | 幻读                                      |
|----------|--------------------------------------|-------------------------------------------|
| 典型操作 | `UPDATE` 已存在的行                  | `INSERT` 或 `DELETE` 匹配范围的行         |
| 变化对象 | 同一行的内容                         | 结果集中的行集合                          |
| 例子     | 员工 1 的工资从 `12000` 变成 `15000` | 工资不低于 `18000` 的员工从 2 人变成 3 人 |

#### 隔离级别

| 隔离级别           | 可避免的问题                     | 并发能力 |
|--------------------|----------------------------------|----------|
| `READ UNCOMMITTED` | 基本不保证隔离，可能发生脏读     | 最高     |
| `READ COMMITTED`   | 避免脏读                         | 较高     |
| `REPEATABLE READ`  | 避免脏读、不可重复读             | 中等     |
| `SERIALIZABLE`     | 隔离程度最高，通常可避免三类问题 | 最低     |

隔离级别越高，一致性通常越好，但锁、等待和并发开销也可能更大。实际行为还取决于数据库的实现方式；面试中应以目标数据库的文档为准。

#### 常见锁

##### 按锁的范围或粒度

| 锁     | 锁住什么                   | 典型特点                               |
|--------|----------------------------|----------------------------------------|
| 表锁   | 整张表                     | 实现简单，但并发能力较低               |
| 行锁   | 一行或若干具体行           | 并发能力较好，不同事务可以修改不同的行 |
| 间隙锁 | 索引记录之间的空档         | 防止其他事务向指定范围插入记录         |
| 临键锁 | 一条索引记录及其前面的间隙 | 某些存储引擎使用的范围锁形式           |
| 列锁   | 某一列                     | 不是常规数据库面试重点，实际支持有限   |

##### 按锁的访问权限

| 锁         | 含义                                   | 常见用途                      |
|------------|----------------------------------------|-------------------------------|
| 共享锁 `S` | 允许读取，通常不允许其他事务修改       | 保护正在读取的数据            |
| 排他锁 `X` | 当前事务修改数据，其他事务通常不能读写 | `UPDATE`、`DELETE` 等修改操作 |
| 意向锁     | 表示事务准备对表中的某些行加锁         | 协调表级锁和行级锁            |

#### 间隙锁与幻读

假设事务执行范围查询：

```sql
SELECT *
FROM employees
WHERE salary >= 18000;
```

如果只锁住查询到的已有行，其他事务仍可能插入一名工资为 `20000` 的员工。事务再次查询时，结果会多出一行，形成幻读。

间隙锁锁住的是索引记录之间的范围，目的是阻止其他事务向该范围插入新记录，因此可以帮助避免这类幻读。

不同数据库的锁模型并不完全相同：

- 某些支持范围锁的存储引擎会使用行锁、间隙锁和临键锁。
- PostgreSQL 主要结合 MVCC 和行级锁；可串行化场景还可能使用谓词锁。
- SQLite 不应按支持行级范围锁的存储引擎理解，主要涉及数据库级或页级锁以及 WAL 机制。

因此，锁的具体行为要以目标数据库文档和执行计划为准。面试中优先掌握：表锁、行锁、共享锁、排他锁、间隙锁和死锁。

#### 悲观锁与乐观并发控制

需要先读后改并阻止其他事务同时修改时，可在支持行锁的数据库中使用 `FOR UPDATE`。下面不是 SQLite 写法，具体加锁范围按目标数据库确认。

```sql
BEGIN;

SELECT id, salary
FROM employees
WHERE id = 1
FOR UPDATE;

UPDATE employees
SET salary = salary + 500
WHERE id = 1;

COMMIT;
```

乐观并发控制常用“版本相同才更新”。下面假设表已经有 `version` 字段，是独立示例，不直接套在前面的练习表上。

```sql
UPDATE employees
SET salary = salary + 500,
    version = version + 1
WHERE id = 1
  AND version = 3;
```

受影响行数为 `1` 表示此次更新成功；为 `0` 可能是记录不存在或版本冲突，需要重读确认。检查与更新必须在同一条语句中完成，不无条件覆盖重试。

### SAVEPOINT：事务内局部回滚

```sql
BEGIN;

UPDATE employees SET salary = salary + 500 WHERE id = 1;
SAVEPOINT salary_updated;

DELETE FROM test_tasks WHERE id = 108;
ROLLBACK TO salary_updated;

COMMIT;
```

`ROLLBACK TO` 只撤销保存点之后的操作，不回滚整个事务。

---

## P1：表结构与约束

> 本节部分语句会修改或删除表结构，均为独立语法示例，不建议直接对练习数据库连续执行。

### 常用约束

| 约束          | 作用                                  |
|---------------|---------------------------------------|
| `PRIMARY KEY` | 唯一标识一行，不能重复且不能为 `NULL` |
| `FOREIGN KEY` | 限制引用值必须存在于关联表中          |
| `NOT NULL`    | 禁止字段为空                          |
| `UNIQUE`      | 禁止字段或字段组合重复                |
| `DEFAULT`     | 插入时未提供字段值则使用默认值        |
| `CHECK`       | 限制字段必须满足指定条件              |

统一示例数据已经使用了常见约束，例如：

```sql
CREATE TABLE departments (
    dept_id INTEGER PRIMARY KEY,
    name VARCHAR(50) NOT NULL UNIQUE
);
```

### 数据库范式

范式用于设计表结构，核心目标是**减少重复数据和更新异常**。实际项目通常先保证结构清晰，再根据查询性能做有限的反范式化。

#### 第一范式（1NF）

每个字段都应该保存不可再拆分的单个值，不能把多个值塞进一个字段。

不推荐把多个电话号码拼在一个字段里：

| employee_id | name | phone_numbers             |
| ----------- | ---- | ------------------------- |
| 1           | 张三   | `13800000000,13900000000` |

这样一个字段实际保存了两个电话号码，不利于单独查询、修改和建立索引。

可以拆成独立的员工电话表：

```text
employees       ：employee_id、name
employee_phones ：employee_id、phone
```

不建议使用逗号拼接、数组或 JSON 保存本应独立查询的多个值。

#### 第二范式（2NF）

满足 1NF，并且**非主键字段完全依赖整个主键**，不能只依赖联合主键的一部分。

例如选课表的联合主键是 `(student_id, course_id)`：

```text
student_id | course_id | student_name | course_name | score
```

其中 `student_name` 只依赖 `student_id`，`course_name` 只依赖 `course_id`，不完全依赖整个联合主键。可以拆成：

```text
students    ：student_id、student_name
courses     ：course_id、course_name
enrollments ：student_id、course_id、score
```

如果表使用单列主键，2NF 的问题通常不明显，但联合主键场景仍要注意。

#### 第三范式（3NF）

满足 2NF，并且**非主键字段不能依赖另一个非主键字段**，也就是消除传递依赖。

例如把部门名称直接重复放在员工表中：

```text
employees：employee_id、name、dept_id、dept_name
```

`dept_name` 实际依赖 `dept_id`，而不是直接依赖员工主键。应拆为当前示例中的结构：

```text
departments：dept_id、name
employees：id、name、dept_id
```

查询部门名称时使用连接：

```sql
SELECT e.name, d.name AS department_name
FROM employees AS e
JOIN departments AS d
    ON e.dept_id = d.dept_id;
```

#### 为什么要做范式化？

主要是减少三类异常：

| 异常     | 含义                                   |
|----------|----------------------------------------|
| 插入异常 | 因为缺少其他无关字段，无法新增一条数据 |
| 更新异常 | 同一信息重复存储，只更新了部分记录     |
| 删除异常 | 删除一条记录时，误删了仍需要保留的信息 |

#### 反范式化

为了减少复杂连接、提高读性能，有时会有意重复或预计算数据，例如在统计表中保存 `employee_count` 或 `average_salary`。

但反范式化会带来同步成本。只有在有明确查询性能需求、并且能通过事务、任务或校验机制维护一致性时才使用。

> 规范化优先保证数据一致性，反范式化是基于性能和读写场景做出的有意识取舍，不是默认把字段重复到各张表。

### ALTER TABLE：修改表结构

以下示例相互独立；修改列名、类型或约束后，其他查询也可能需要调整。

```sql
-- 新增列
ALTER TABLE employees
ADD COLUMN email TEXT;
```

```sql
-- 重命名列
ALTER TABLE employees
RENAME COLUMN name TO employee_name;
```

```sql
-- 删除列，同时删除该列的数据
ALTER TABLE employees
DROP COLUMN email;
```

### DROP 与 TRUNCATE

```sql
-- 删除表结构及全部数据
DROP TABLE old_tasks;
```

```sql
-- 快速清空表，但保留表结构
TRUNCATE TABLE temporary_tasks;
```

区别：

- `DELETE FROM table_name`：逐行删除，可带 `WHERE`，通常记录更多事务日志。
- `TRUNCATE TABLE`：清空整表，不支持 `WHERE`；SQLite 没有该语法，可使用 `DELETE FROM table_name`。
- `DROP TABLE`：表结构和数据一起删除。

### VIEW：保存查询逻辑

- 视图通常保存查询定义，本身不存储普通查询结果。
- 适合复用复杂查询、限制暴露字段和统一查询入口。

```sql
CREATE VIEW employee_department_view AS
SELECT
    e.id,
    e.name,
    e.salary,
    d.name AS department_name
FROM employees AS e
LEFT JOIN departments AS d
    ON e.dept_id = d.dept_id;
```

```sql
SELECT *
FROM employee_department_view
WHERE salary >= 18000;
```

---

## P1：索引与性能基础

### INDEX：减少全表扫描

- 适合经常出现在 `WHERE`、`JOIN`、`ORDER BY` 中且区分度较高的列。
- 索引会占用空间，并增加 `INSERT`、`UPDATE`、`DELETE` 成本，不是越多越好。

```sql
CREATE INDEX idx_tasks_employee_status
ON test_tasks (employee_id, status);
```

联合索引 `(employee_id, status)` 通常更适合以下查询：

```sql
SELECT *
FROM test_tasks
WHERE employee_id = 1
  AND status = 'passed';
```

#### 联合索引的常见理解

索引 `(a, b, c)` 通常优先支持从左开始的条件，例如：

- `a`
- `a, b`
- `a, b, c`

只按 `b` 或 `c` 查询通常无法充分使用该索引，具体仍以执行计划为准。

### EXPLAIN：查看执行计划

```sql
EXPLAIN
SELECT *
FROM test_tasks
WHERE employee_id = 1
  AND status = 'passed';
```

重点观察：

- 是否全表扫描。
- 是否使用预期索引。
- 预计扫描行数是否过大。
- 连接顺序是否合理。
- 是否出现额外排序或临时表。

#### 常见优化原则

- 不读取不需要的列，避免无意义的 `SELECT *`。
- 先过滤再连接或聚合，减少中间数据量。
- 避免在索引列上进行不必要的函数计算。
- 不用 `DISTINCT` 掩盖错误连接造成的重复数据。
- 性能问题以执行计划和实际数据量为依据，不凭感觉加索引。

---

## P2：常用补充

### UNION 与 UNION ALL

- `UNION` 合并结果并去重。
- `UNION ALL` 保留重复行，通常更快。
- 两边查询的列数和对应类型需要兼容。

```sql
SELECT name FROM employees WHERE dept_id = 1
UNION ALL
SELECT name FROM employees WHERE dept_id = 2;
```

### INTERSECT 与 EXCEPT

- `INTERSECT`：返回两个结果集的交集。
- `EXCEPT`：返回第一个结果集中存在、第二个结果集中不存在的数据。
- 和 `UNION` 一样，两侧列数及对应类型需要兼容。

```sql
-- 同时参与 Alpha 和 Beta 项目的员工
SELECT employee_id FROM test_tasks WHERE project = 'Alpha'
INTERSECT
SELECT employee_id FROM test_tasks WHERE project = 'Beta';
```

```sql
-- 参与 Alpha 但没有参与 Beta 的员工
SELECT employee_id FROM test_tasks WHERE project = 'Alpha'
EXCEPT
SELECT employee_id FROM test_tasks WHERE project = 'Beta';
```

> 并非所有数据库都支持 `INTERSECT` 和 `EXCEPT`；不支持时可以使用连接或 `EXISTS / NOT EXISTS` 改写。

### CAST：显式类型转换

```sql
SELECT
    name,
    CAST(salary AS TEXT) AS salary_text
FROM employees;
```

计算比例时，也可以将整数转换为小数，避免某些数据库执行整数除法：

```sql
SELECT
    project,
    CAST(SUM(CASE WHEN status = 'passed' THEN 1 ELSE 0 END) AS REAL)
        / COUNT(*) AS pass_rate
FROM test_tasks
GROUP BY project;
```

### 字符串函数

不同数据库函数名可能不同，常见能力包括：

```sql
SELECT
    UPPER(project) AS upper_project,
    LENGTH(project) AS name_length,
    project || '-' || status AS task_label
FROM test_tasks;
```

> 字符串拼接函数存在方言差异；SQLite / PostgreSQL 常用 `||`，其他数据库请查对应文档。

### 日期范围

时间字段包含时分秒时，推荐使用左闭右开范围：

```sql
SELECT *
FROM test_tasks
WHERE finished_at >= '2025-05-01'
  AND finished_at <  '2025-06-01';
```

这样通常比把时间列转换成字符串更容易利用索引。

### 查找重复数据

```sql
-- 查找重复的“员工 + 项目”组合
SELECT employee_id, project, COUNT(*) AS duplicate_count
FROM test_tasks
GROUP BY employee_id, project
HAVING COUNT(*) > 1;
```

### 删除重复数据的思路

先用窗口函数编号，再保留每组第一条。删除语法因数据库不同而不同，笔试中通常写出识别重复行的查询即可：

```sql
WITH numbered AS (
    SELECT
        id,
        ROW_NUMBER() OVER (
            PARTITION BY employee_id, project
            ORDER BY id
        ) AS row_num
    FROM test_tasks
)
SELECT id
FROM numbered
WHERE row_num > 1;
```

---

## 必须掌握的典型题

### 每个部门工资最高的员工

```sql
WITH ranked AS (
    SELECT
        e.*,
        DENSE_RANK() OVER (
            PARTITION BY dept_id
            ORDER BY salary DESC
        ) AS salary_rank
    FROM employees AS e
    WHERE dept_id IS NOT NULL
)
SELECT name, dept_id, salary
FROM ranked
WHERE salary_rank = 1;
```

### 工资高于所在部门平均值的员工

```sql
SELECT e.name, e.dept_id, e.salary
FROM employees AS e
JOIN (
    SELECT dept_id, AVG(salary) AS average_salary
    FROM employees
    WHERE dept_id IS NOT NULL
    GROUP BY dept_id
) AS department_stats
    ON e.dept_id = department_stats.dept_id
WHERE e.salary > department_stats.average_salary;
```

窗口函数写法更紧凑：

```sql
WITH employee_stats AS (
    SELECT
        e.*,
        AVG(salary) OVER (PARTITION BY dept_id) AS average_salary
    FROM employees AS e
)
SELECT name, dept_id, salary
FROM employee_stats
WHERE salary > average_salary;
```

### 从未执行失败任务的员工

```sql
SELECT e.name
FROM employees AS e
WHERE NOT EXISTS (
    SELECT 1
    FROM test_tasks AS t
    WHERE t.employee_id = e.id
      AND t.status = 'failed'
);
```

### 每个项目的通过率

```sql
SELECT
    project,
    COUNT(*) AS total_count,
    SUM(CASE WHEN status = 'passed' THEN 1 ELSE 0 END) AS passed_count,
    ROUND(
        100.0 * SUM(CASE WHEN status = 'passed' THEN 1 ELSE 0 END)
        / COUNT(*),
        2
    ) AS pass_rate_percent
FROM test_tasks
GROUP BY project;
```

### 第二高工资

```sql
WITH ranked AS (
    SELECT
        name,
        salary,
        DENSE_RANK() OVER (ORDER BY salary DESC) AS salary_rank
    FROM employees
)
SELECT name, salary
FROM ranked
WHERE salary_rank = 2;
```

---

## 易错点速查

- `NULL` 使用 `IS NULL`，不要写 `= NULL`。
- `COUNT(*)` 统计行；`COUNT(column)` 忽略该列的 `NULL`。
- `WHERE` 过滤分组前的数据；`HAVING` 过滤聚合后的组。
- `LEFT JOIN` 的右表过滤条件放错到 `WHERE`，可能意外变成内连接效果。
- 一对多连接出现多行通常是正常展开，不要先用 `DISTINCT` 掩盖。
- `NOT IN` 遇到子查询中的 `NULL` 可能产生意外结果，优先考虑 `NOT EXISTS`。
- 求排名并保留并列时使用 `RANK` 或 `DENSE_RANK`；只要固定 N 行时常用 `ROW_NUMBER`。
- 整数除法可能丢失小数，使用 `100.0` 或显式类型转换。
- 分页和 Top N 必须配合稳定的 `ORDER BY`，否则结果顺序不可靠。
- `UPDATE`、`DELETE` 前先用相同条件执行 `SELECT`。

---

## 练习方法

每道题按以下步骤处理：

1. 明确最终结果“一行代表什么”。
2. 确认需要哪些表及其关联键。
3. 先写 `FROM / JOIN`，检查基础数据是否重复。
4. 再加 `WHERE` 过滤原始行。
5. 需要汇总时再加 `GROUP BY / HAVING`。
6. 需要组内排名或保留明细时考虑窗口函数。
7. 最后补充 `SELECT`、`ORDER BY` 和 `LIMIT`。

### 面试前最低练习量

- 基础筛选与排序：5 题。
- `GROUP BY / HAVING`：5 题。
- 多表连接：5 题。
- 子查询与 `EXISTS`：3 题。
- 窗口函数：5 题。
- 综合限时题：5 题，每题控制在 10～15 分钟。

完成后应能在不查资料的情况下写出：

- 每组最大值、第二高值和 Top N。
- 没有关联记录的数据。
- 高于组内平均值的数据。
- 按条件统计数量、通过率和占比。
- 查找重复数据和连续记录的基本思路。
