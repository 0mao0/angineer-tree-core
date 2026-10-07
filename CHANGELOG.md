# Changelog

## 0.1.0（对外发布基线）

首个对外发布版本：AnGIneer 的通用树节点存储层从主仓库（`services/tree-core`）独立成包。
纯 Python 库，落在调用方传入的一个 SQLite 连接上，只做存储与排序，不含 HTTP 与业务语义。

### 能力
- **节点 CRUD**：`insert_node` / `get_node` / `update_node` / `delete_node`（删节点连带整棵子树并重排原父级）；
- **移动与排序**：`move_node` 跨父节点搬迁时自动补 `sort_order` 并归一化新旧两组兄弟；`normalize_siblings` 把同级顺序收敛成 `0..n-1`；
- **作用域隔离**：`tree_type` + `scope_id` 两个隔离键，同一张 `tree_node` 表可并存多棵树 × 多个作用域（`list_children` / `list_nodes_by_scope` / `list_nodes_by_type`）；
- **软删标记**：`mark_node_deleted` / `is_node_deleted`（标记不自动 commit，事务边界留给调用方）；
- **契约模型**：`TreeNodeData` / `MoveNodeRequest`（pydantic v2），供消费方做入参校验；
- **extra 字段**：任意字典存 `extra_json`，读回自动还原为 `extra`。

### 发布前修掉的两处内部缺陷（本包首次对外，记入基线）
- `_next_sort_order` 曾用 `row[0] or -1` 判断"同级还没有节点"，把合法值 `0` 当成"没有"——导致**同级第二个及以后的节点 sort_order 全是 0**（对外回归测试抓出）。现改为显式判 `None`；受影响的历史数据可用 `normalize_siblings` 一次性收敛。
- `is_node_deleted` 对 `sqlite3.Row` 调用 `.get()`（该类型没有此方法），节点存在时必抛 `AttributeError`。现改为列名下标取值。

### 使用前提（已在 README 写明）
- 连接与事务归调用方；`init_table` 会确保连接按列名取行（`row_factory = sqlite3.Row`），调用方无需自行设置。

### 开发与测试
- `tests/test_tree_store.py` 12 例：建表幂等与 row_factory、插入排序递增、子节点插入序、跨父移动与双边归一化、缺口压缩、子树删除与原级重排、软删未提交可回滚、更新白名单、作用域隔离、契约模型校验。
