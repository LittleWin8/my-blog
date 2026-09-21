+++
title = "树形结构通用工具类"
date = 2026-05-10
draft = false
tags = ["算法", "Java", "毕业设计"]
+++
> 在毕设项目的开发过程中，我遇到了一个反复出现的场景：菜单需要树形结构、分类需要树形结构、以后如果有组织架构大概率也需要。每次写一遍递归构建逻辑显然不优雅，于是我封装了一个通用的 `TreeUtils` 工具类。这篇文章记录我的设计思路和实现细节。

# 一、为什么需要通用树构建

在管理系统中，树形结构几乎无处不在：

- **菜单树**：后台管理的左侧导航，多级嵌套
- **分类树**：笔记的分类体系，支持父子关系
- **组织架构树**：如果将来扩展到团队协作场景

这些场景的数据模型是一致的：每个节点有 `id`、`parentId`、`children`，数据库存的是扁平列表，前端需要的是嵌套树。

如果每个场景都写一遍构建逻辑，代码会大量重复。核心思路是：**定义一个通用接口，让所有需要树形结构的实体实现它，工具类只依赖接口操作**。

# 二、核心设计：TreeNode 接口

第一步是抽象出树节点的共性。我定义了一个泛型接口 `TreeNode<T, K>`：

```java
public interface TreeNode<T, K> {

    K getId();                  // 节点ID
    K getParentId();            // 父节点ID
    List<T> getChildren();      // 子节点列表
    void setChildren(List<T> children);

    // 便捷方法
    default void addChild(T child) { ... }
    default boolean hasChildren() { ... }
    default int childrenCount() { ... }
    default boolean isLeaf() { ... }
}

```

两个泛型参数的设计：

- `T`：节点自身的类型（通常是实现类自身），用于 `children` 列表的类型约束
- `K`：ID 的类型，支持 `Long`、`String`、`Integer` 等，不用硬编码

这样任何实体只需要 `implements TreeNode<自身类型, ID类型>` 就能接入工具类。项目中有三个实现：

| 实体 | 用途 | 泛型参数 |
| --- | --- | --- |
| `SysCategory` | 笔记分类树 | `TreeNode<SysCategory, Long>` |
| `MenuVO` | 管理端菜单路由树 | `TreeNode<MenuVO, Long>` |
| `MenuTreeVO` | 角色权限分配的菜单树 | `TreeNode<MenuTreeVO, Long>` |

以 `SysCategory` 为例，实现非常简洁：

```java
@Data
@TableName("sys_category")
public class SysCategory implements TreeNode<SysCategory, Long> {

    @TableId(value = "category_id", type = IdType.AUTO)
    private Long categoryId;

    private String name;
    private Long parentId;
    private Integer sortOrder;
    private Integer status;

    @TableField(exist = false)               // 非数据库字段
    @JsonInclude(JsonInclude.Include.NON_EMPTY)  // 空子节点不序列化
    private List<SysCategory> children;

    @Override
    public Long getId() { return this.categoryId; }

    @Override
    public Long getParentId() { return this.parentId; }

    @Override
    public List<SysCategory> getChildren() { return this.children; }

    @Override
    public void setChildren(List<SysCategory> children) { this.children = children; }
}

```

关键点：`children` 字段标记了 `@TableField(exist = false)`，它不会从数据库读取，而是在查询出扁平列表后由 `TreeUtils.build()` 动态填充。

# 三、构建树：O(n) 的 Map 索引法

核心的 `build()` 方法是整个工具类的灵魂。先看一下调用方式：

```java
// 分类树：按 sortOrder 排序
List<SysCategory> tree = TreeUtils.build(allCategories, 0L,
        Comparator.comparingInt(SysCategory::getSortOrder));

// 菜单树：不排序
List<MenuVO> menuTree = TreeUtils.build(menuVOList, 0L);

```

`rootParentId` 参数（这里是 `0L`）代表根节点的父 ID——也就是数据库中顶级节点的 `parent_id` 字段值。

## 3.1 核心思想：先建索引，再挂载

传统做法是双重循环——对每个节点遍历整个列表找它的父节点，时间复杂度 O(n²)。这个算法的思路是 **先用 Map 建索引，再遍历一次挂载**，把复杂度降到 O(n)。

用一个具体例子走一遍。假设数据库查出 5 条分类数据：

| ID | NAME | PARENTID |
| --- | --- | --- |
| 1 | 技术 | 0 |
| 2 | Java | 1 |
| 3 | Spring | 2 |
| 4 | 生活 | 0 |
| 5 | 美食 | 4 |

**第一步：建索引**

用所有节点的 id 作为 key，构建 `Map<id, 节点对象>`：

```
Map = {
    1 → {id:1, name:"技术", parentId:0},
    2 → {id:2, name:"Java", parentId:1},
    3 → {id:3, name:"Spring", parentId:2},
    4 → {id:4, name:"生活", parentId:0},
    5 → {id:5, name:"美食", parentId:4}
}
```

作用：以后想找 id=2 的节点，直接 `map.get(2)`，O(1) 就能拿到。

**第二步：遍历一次，全部挂好**

```
当前节点     parentId    动作
技术(1)      0           parentId==0 → 加入 roots
Java(2)      1           map.get(1)=技术 → 挂到技术.children
Spring(3)    2           map.get(2)=Java → 挂到Java.children
生活(4)      0           parentId==0 → 加入 roots
美食(5)      4           map.get(4)=生活 → 挂到生活.children
```

**第三步：看最终结果**

```
roots = [技术, 生活]
​
技术
  └── Java
        └── Spring
​
生活
  └── 美食
```

**为什么快？**

| 方法 | 时间复杂度 | 1000 条数据的操作次数 |
| --- | --- | --- |
| 传统双重循环 | O(n²) | 约 100 万次 |
| Map 索引法 | O(n) | 约 1000 次 |

## 3.2 代码实现

对应到 `TreeUtils.build()` 的源码：

```java
public static <T extends TreeNode<T, K>, K> List<T> build(
        List<T> list, K rootParentId, Comparator<T> comparator) {

    if (list == null || list.isEmpty()) return new ArrayList<>();

    // 1. 过滤 null 元素
    List<T> filteredList = list.stream()
            .filter(Objects::nonNull)
            .collect(Collectors.toList());

    // 2. 构建 ID -> 节点的映射（核心：O(1) 查找）
    Map<K, T> idToNodeMap = filteredList.stream()
            .collect(Collectors.toMap(
                    TreeNode::getId,
                    node -> node,
                    (existing, replacement) -> existing,  // ID 冲突保留第一个
                    LinkedHashMap::new                     // 保持插入顺序
            ));

    List<T> roots = new ArrayList<>();

    // 3. 单次遍历，挂载子节点
    for (T node : filteredList) {
        K parentId = node.getParentId();

        if (Objects.equals(parentId, rootParentId)) {
            roots.add(node);  // 根节点
        } else if (parentId != null) {
            T parent = idToNodeMap.get(parentId);
            if (parent != null) {
                if (parent.getChildren() == null) {
                    parent.setChildren(new ArrayList<>());
                }
                parent.getChildren().add(node);
            }
            // parent 不存在说明数据不完整，记录 debug 日志即可
        }
    }

    // 4. 可选排序
    if (comparator != null) {
        sortTree(roots, comparator);
    }

    return roots;
}

```

**时间复杂度分析**：

- 建 Map 索引：O(n)
- 遍历挂载：O(n)
- 排序：取决于 Comparator，通常是 O(n log n)
- 总体：**O(n log n)**，相比朴素 O(n²) 有本质提升

几个细节处理：

- `LinkedHashMap` 保持插入顺序，这样不排序时也能保证结果稳定
- **ID 冲突策略** `(existing, replacement) -> existing`，保留第一个，避免意外覆盖
- **null 元素过滤**，防止数据库查询返回 null 导致 NPE

## 3.3 递归排序

排序是独立于构建的，支持对整棵树的每一层排序：

```java
private static <T extends TreeNode<T, K>, K> void sortTree(
        List<T> nodes, Comparator<T> comparator) {
    if (nodes == null || nodes.isEmpty()) return;
    nodes.sort(comparator);
    for (T node : nodes) {
        if (node.getChildren() != null) {
            sortTree(node.getChildren(), comparator);  // 递归子层
        }
    }
}

```

实际使用时传入 `Comparator.comparingInt(SysCategory::getSortOrder)`，就能实现每层分类按 `sort_order` 字段排序。

# **四、项目中的实际调用**

项目中 `TreeUtils` 被 6 处业务代码调用，用到了两个方法：`build()` 和 `findAllChildIds()`。

### 4.1 `build()` — 构建树形结构（4 处）

**管理端菜单路由**（[AdminAuthServiceImpl.java:225](https://github.com/LittleWin8/GraduationProject/blob/main/smart-note-system/system/src/main/java/com/littlewin/system/service/impl/AdminAuthServiceImpl.java#L225)）

```java
// 调用工具类直接构建树形结构 (假设根节点的 parentId 是 0L)
return TreeUtils.build(menuVOList, 0L);
```

**角色权限菜单树**（[SysRoleController.java:163](https://github.com/LittleWin8/GraduationProject/blob/main/smart-note-system/system/src/main/java/com/littlewin/system/controller/SysRoleController.java#L163)）

```java
// 3. 使用 TreeUtils 构建树形结构 (根节点 parentId 为 0L)
List<MenuTreeVO> tree = TreeUtils.build(flatList, 0L);
return Result.success(tree);
```

**管理端分类树（带排序）**（[AdminCategoryServiceImpl.java:42](https://github.com/LittleWin8/GraduationProject/blob/main/smart-note-system/note/src/main/java/com/littlewin/note/service/impl/AdminCategoryServiceImpl.java#L42)）

```java
List<SysCategory> all = categoryMapper.selectList(wrapper);
return TreeUtils.build(all, 0L, Comparator.comparingInt(SysCategory::getSortOrder));
```

**小程序端分类树**（[WxCategoryController.java:31](https://github.com/LittleWin8/GraduationProject/blob/main/smart-note-system/note/src/main/java/com/littlewin/note/controller/WxCategoryController.java#L31)）

```java
List<SysCategory> activeList = categoryMapper.selectList(wrapper);
// 2. 构建树形结构
List<SysCategory> tree = TreeUtils.build(activeList, 0L);
return Result.success(tree);
```

## 4.2 `findAllChildIds()` — 获取子树 ID 集合（2 处）

用户在小程序端选了"技术笔记"这个父分类，需要查出它和所有子分类下的笔记。两处调用写法一致：

**笔记列表按分类筛选**（[WxNoteServiceImpl.java:197](https://github.com/LittleWin8/GraduationProject/blob/main/smart-note-system/note/src/main/java/com/littlewin/note/service/impl/WxNoteServiceImpl.java#L197)）

```java
private List<Long> getAllCategoryIds(Long categoryId) {
    List<SysCategory> allCategories = sysCategoryMapper.selectList(null);
    return TreeUtils.findAllChildIds(allCategories, categoryId, true);
}
```

**统计数据按分类聚合**（[WxNoteStatsServiceImpl.java:106](https://github.com/LittleWin8/GraduationProject/blob/main/smart-note-system/note/src/main/java/com/littlewin/note/service/impl/WxNoteStatsServiceImpl.java#L106)）

```java
// 参数：全量列表, 目标起始ID, 是否包含自身
return TreeUtils.findAllChildIds(allCategories, categoryId, true);
```

共同模式：先一次性查出所有分类，再调用工具类提取子树 ID。`includeSelf = true` 表示结果包含传入的 `categoryId` 本身。

一个工具类支撑了菜单、分类、笔记查询三个核心模块，体现了通用封装的价值。

# 五、设计反思

**做得好的地方**：

- 泛型设计让工具类完全不依赖具体业务实体，可移植性强
- Map 索引法保证了性能
- 空值防护贯穿所有方法，不会因为脏数据就 NPE
- 工具类还预置了 `flatten`（扁平化）、`findPath`（路径追溯）、`findOrphanNodes`（孤儿检测）等扩展方法，为后续需求预留

**可以改进的地方**：

- 目前只支持单根（一个 `rootParentId`），如果有多棵树的场景需要多次调用
- `findAllChildIds` 每次调用都重新建分组 Map，高频调用时可以考虑缓存
- 没有支持懒加载（按层查询），对于深层大树可能需要

**如果重新做**，我可能会考虑引入 `@TreeId`、`@TreeParent` 注解来自动映射字段，而不是要求实现接口。但对于毕设的复杂度来说，当前方案已经足够——简洁、实用、好理解。

---

> 这个工具类的核心价值在于：**用一个接口约束 + 一个工具类，统一了所有树形结构的处理逻辑**。不需要为菜单写一套、为分类再写一套。这种"面向接口编程 + 泛型抽象"的思路，是我在毕设开发过程中最大的收获之一。

# **附录：完整源码**

## TreeNode.java

```java
package com.littlewin.common.core;

import java.util.ArrayList;
import java.util.List;

/**
 * 树节点接口
 *
 * @param <T> 节点类型（通常是实现类自身）
 * @param <K> ID类型（如 Long, String, Integer）
 */
public interface TreeNode<T, K> {

    K getId();

    K getParentId();

    List<T> getChildren();

    void setChildren(List<T> children);

    default void addChild(T child) {
        if (getChildren() == null) {
            setChildren(new ArrayList<>());
        }
        getChildren().add(child);
    }

    default boolean hasChildren() {
        return getChildren() != null && !getChildren().isEmpty();
    }

    default int childrenCount() {
        return hasChildren() ? getChildren().size() : 0;
    }

    default boolean isLeaf() {
        return !hasChildren();
    }
}
```

## TreeUtils.java

```java
package com.littlewin.common.utils;

import com.littlewin.common.core.TreeNode;
import lombok.extern.slf4j.Slf4j;

import java.util.*;
import java.util.stream.Collectors;

@Slf4j
public class TreeUtils {

    private TreeUtils() {
    }

    /**
     * 构建嵌套树形结构
     */
    public static <T extends TreeNode<T, K>, K> List<T> build(List<T> list, K rootParentId) {
        return build(list, rootParentId, null);
    }

    /**
     * 构建嵌套树形结构并排序
     */
    public static <T extends TreeNode<T, K>, K> List<T> build(
            List<T> list, K rootParentId, Comparator<T> comparator) {

        if (list == null || list.isEmpty()) return new ArrayList<>();

        List<T> filteredList = list.stream()
                .filter(Objects::nonNull)
                .collect(Collectors.toList());

        if (filteredList.isEmpty()) return new ArrayList<>();

        Map<K, T> idToNodeMap = filteredList.stream()
                .collect(Collectors.toMap(
                        TreeNode::getId,
                        node -> node,
                        (existing, replacement) -> existing,
                        LinkedHashMap::new
                ));

        List<T> roots = new ArrayList<>();

        for (T node : filteredList) {
            K parentId = node.getParentId();

            if (Objects.equals(parentId, rootParentId)) {
                roots.add(node);
            } else if (parentId != null) {
                T parent = idToNodeMap.get(parentId);
                if (parent != null) {
                    if (parent.getChildren() == null) {
                        parent.setChildren(new ArrayList<>());
                    }
                    parent.getChildren().add(node);
                } else {
                    log.debug("Parent node not found for node id: {}, parentId: {}",
                            node.getId(), parentId);
                }
            }
        }

        if (comparator != null) {
            sortTree(roots, comparator);
        }

        return roots;
    }

    /**
     * 获取指定节点下的所有子 ID 集合（不含自身）
     */
    public static <T extends TreeNode<T, K>, K> List<K> findAllChildIds(
            List<T> all, K parentId) {
        return findAllChildIds(all, parentId, false);
    }

    /**
     * 获取指定节点下的所有子 ID 集合
     */
    public static <T extends TreeNode<T, K>, K> List<K> findAllChildIds(
            List<T> all, K parentId, boolean includeSelf) {

        if (all == null || all.isEmpty() || parentId == null) {
            return includeSelf && parentId != null
                    ? Collections.singletonList(parentId)
                    : Collections.emptyList();
        }

        Map<K, List<T>> parentToChildrenMap = all.stream()
                .filter(Objects::nonNull)
                .filter(node -> node.getParentId() != null)
                .collect(Collectors.groupingBy(
                        TreeNode::getParentId,
                        Collectors.toList()
                ));

        List<K> result = new ArrayList<>();
        collectChildIds(parentId, parentToChildrenMap, result);

        if (includeSelf) {
            result.add(0, parentId);
        }

        return result;
    }

    /**
     * 获取指定节点下的所有子节点（扁平化列表）
     */
    public static <T extends TreeNode<T, K>, K> List<T> findAllChildNodes(
            List<T> all, K parentId) {

        if (all == null || all.isEmpty() || parentId == null) {
            return new ArrayList<>();
        }

        Map<K, List<T>> parentToChildrenMap = all.stream()
                .filter(Objects::nonNull)
                .filter(node -> node.getParentId() != null)
                .collect(Collectors.groupingBy(TreeNode::getParentId));

        List<T> result = new ArrayList<>();
        collectChildNodes(parentId, parentToChildrenMap, result);
        return result;
    }

    /**
     * 扁平化树结构（栈迭代，深度优先）
     */
    public static <T extends TreeNode<T, K>, K> List<T> flatten(List<T> tree) {
        List<T> result = new ArrayList<>();
        if (tree == null || tree.isEmpty()) return result;

        Deque<T> stack = new ArrayDeque<>();
        for (int i = tree.size() - 1; i >= 0; i--) {
            stack.push(tree.get(i));
        }

        while (!stack.isEmpty()) {
            T node = stack.pop();
            result.add(node);
            if (node.getChildren() != null && !node.getChildren().isEmpty()) {
                List<T> children = node.getChildren();
                for (int i = children.size() - 1; i >= 0; i--) {
                    stack.push(children.get(i));
                }
            }
        }
        return result;
    }

    /**
     * 查找从根节点到指定节点的路径
     */
    public static <T extends TreeNode<T, K>, K> List<T> findPath(
            List<T> all, K nodeId) {

        if (all == null || all.isEmpty() || nodeId == null) {
            return new ArrayList<>();
        }

        Map<K, T> idToNodeMap = all.stream()
                .filter(Objects::nonNull)
                .collect(Collectors.toMap(TreeNode::getId, node -> node, (v1, v2) -> v1));

        T target = idToNodeMap.get(nodeId);
        if (target == null) return new ArrayList<>();

        LinkedList<T> path = new LinkedList<>();
        T current = target;
        while (current != null) {
            path.addFirst(current);
            K parentId = current.getParentId();
            if (parentId == null) break;
            current = idToNodeMap.get(parentId);
        }
        return path;
    }

    /**
     * 验证树结构完整性（检查孤儿节点）
     */
    public static <T extends TreeNode<T, K>, K> List<T> findOrphanNodes(
            List<T> all, K rootParentId) {

        if (all == null || all.isEmpty()) return new ArrayList<>();

        Set<K> allIds = all.stream()
                .filter(Objects::nonNull)
                .map(TreeNode::getId)
                .filter(Objects::nonNull)
                .collect(Collectors.toSet());

        return all.stream()
                .filter(Objects::nonNull)
                .filter(node -> {
                    K parentId = node.getParentId();
                    return parentId != null
                            && !Objects.equals(parentId, rootParentId)
                            && !allIds.contains(parentId);
                })
                .collect(Collectors.toList());
    }

    // ==================== 私有辅助方法 ====================

    /**
     * 收集子节点ID
     */
    private static <T extends TreeNode<T, K>, K> void collectChildIds(
            K parentId, Map<K, List<T>> parentMap, List<K> result) {

        List<T> children = parentMap.get(parentId);
        if (children != null) {
            for (T child : children) {
                K childId = child.getId();
                if (childId != null) {
                    result.add(childId);
                    collectChildIds(childId, parentMap, result);
                }
            }
        }
    }

    /**
     * 收集子节点对象
     */
    private static <T extends TreeNode<T, K>, K> void collectChildNodes(
            K parentId, Map<K, List<T>> parentMap, List<T> result) {

        List<T> children = parentMap.get(parentId);
        if (children != null) {
            for (T child : children) {
                result.add(child);
                collectChildNodes(child.getId(), parentMap, result);
            }
        }
    }

    /**
     * 递归排序树结构
     */
    private static <T extends TreeNode<T, K>, K> void sortTree(
            List<T> nodes, Comparator<T> comparator) {

        if (nodes == null || nodes.isEmpty()) return;
        nodes.sort(comparator);
        for (T node : nodes) {
            if (node.getChildren() != null && !node.getChildren().isEmpty()) {
                sortTree(node.getChildren(), comparator);
            }
        }
    }
}
```
