---
title:  'MySQL 中执行计划分析 - Optimizer trace表'
date:  2021-12-21
tags: [mysql]
categories: [数据库,mysql]
---

## 1. 概述

​		对于 MySQL 5.6 以及之前的版本来说，查询优化器就像是一个黑盒子一样，你只能通过 EXPLAIN 语句查看到最后优化器决定使用的执行计划，却无法知道它为什么做这个决策。

​		在 MySQL 5.6 以及之后的版本中，MySQL 提出了一个 optimizer trace 的功能，这个功能可以让我们方便的查看优化器生成执行计划的整个过程。

## 2.Optimizer trace的使用

> Optimizer trace 并不是自动就会默认开启的，开启 trace 多多少少都会有一些额外的工作要做，因此并不建议一直开着。但 trace 属于轻量级的工具，开启和关闭都非常简便，对系统的影响也微乎其微。而且支持在 session 中开启，不影响其它 session，对系统的影响降到了最低。

### 2.1 查看`optimizer trace`状态

```sql
> show variables like '%optimizer_trace%';

+----------------------------+--------------------------------------------------------------------------+
|Variable_name               |Value                                                                     |
+----------------------------+--------------------------------------------------------------------------+
|optimizer_trace             |enabled=off,one_line=off                                                  |
|optimizer_trace_features    |greedy_search=on,range_optimizer=on,dynamic_range=on,repeated_subselect=on|
|optimizer_trace_limit       |1                                                                         |
|optimizer_trace_max_mem_size|16384                                                                     |
|optimizer_trace_offset      |-1                                                                        |
+----------------------------+--------------------------------------------------------------------------+
```

- optimizer_trace：enabled状态；one_line的值是控制输出格式的，如果为on那么所有输出都将在一行中展示
- optimizer_trace_limit：OPTIMIZER_TRACE表中保存条数
- optimizer_trace_max_mem_size：由于优化过程可能会输出很多，如果超过某个限制时，多余的文本将不会被显示
- optimizer_trace_offset：查询OPTIMIZER_TRACE表时的偏移量

###  2.2 开启`optimizer trace`功能

```sql
SET optimizer_trace="enabled=on";
SET optimizer_trace_limit=10;
SET optimizer_trace_offset=10;
SET optimizer_trace_max_mem_size = 32768;
```

注意：在这里设置了optimizer_trace_limit为10主要是因为在使用`DataGrip`时会自动插入多条数据影响查看

### 2.3  **查询上一个语句的优化过程**

```sql
SELECT * FROM information_schema.OPTIMIZER_TRACE;
```

- QUERY ：表示我们的查询语句。
- TRACE ：表示优化过程的JSON格式文本。
- MISSING_BYTES_BEYOND_MAX_MEM_SIZE ：由于优化过程可能会输出很多，如果超过某个限制时，多余的文本将不会被显示，这个字段展示了被忽略的文本字节数。
- INSUFFICIENT_PRIVILEGES ：表示是否没有权限查看优化过程，默认值是0，只有某些特殊情况下才会是1 ，我们暂时不关心这个字段的值

### 2.4  关闭optimizer trace 功能

```sql
SET optimizer_trace="enabled=off";
SET optimizer_trace_limit=1;
SET optimizer_trace_offset=-1;
SET optimizer_trace_max_mem_size = 16384;
```

## 3. 具体分析

TRACE文本分析：

```json
{
  "steps": [
    {
      "join_preparation": { #  prepare阶段
        "select#": 1,
        "steps": [
          {
            "expanded_query": "/* select#1 */ select sql_no_cache `item_sale_summary`.`ent_id` AS `ent_id`,`item_sale_summary`.`region_code` AS `region_code`,ceiling((count(distinct `item_sale_summary`.`item_code`,`item_sale_summary`.`barcode`,date_format(`item_sale_summary`.`trans_date`,'%Y-%m-%d')) / ((to_days('2021-12-05') - to_days('2021-11-05')) + 1))) AS `sku_item_sale` from `item_sale_summary` where ((`item_sale_summary`.`trans_date` between '2021-11-01' and '2021-11-15') and (`item_sale_summary`.`ent_id` = 1747964630024192400)) group by `item_sale_summary`.`ent_id`,`item_sale_summary`.`region_code`"
          }
        ]
      }
    },
    {
      "join_optimization": { # optimize阶段
        "select#": 1,
        "steps": [
          {
            "condition_processing": { # 处理搜索条件
              "condition": "WHERE",
              # 原始搜索条件
              "original_condition": "((`item_sale_summary`.`trans_date` between '2021-11-01' and '2021-11-15') and (`item_sale_summary`.`ent_id` = 1747964630024192400))",
              "steps": [
                {  
                  "transformation": "equality_propagation", # 等值传递转换
                  "resulting_condition": "((`item_sale_summary`.`trans_date` between '2021-11-01' and '2021-11-15') and multiple equal(1747964630024192400, `item_sale_summary`.`ent_id`))"
                },
                {
                  "transformation": "constant_propagation",  # 常量传递转换 
                  "resulting_condition": "((`item_sale_summary`.`trans_date` between '2021-11-01' and '2021-11-15') and multiple equal(1747964630024192400, `item_sale_summary`.`ent_id`))"
                },
                {
                  "transformation": "trivial_condition_removal",  # 去除没用的条件
                  "resulting_condition": "((`item_sale_summary`.`trans_date` between '2021-11-01' and '2021-11-15') and multiple equal(1747964630024192400, `item_sale_summary`.`ent_id`))"
                }
              ]
            }
          },
          {
            "substitute_generated_columns": {  # 替换虚拟生成列
            }
          },
          {
            "table_dependencies": [  # 表的依赖信息
              {
                "table": "`item_sale_summary`",
                "row_may_be_null": false,
                "map_bit": 0,
                "depends_on_map_bits": [
                ]
              }
            ]
          },
          {
            "ref_optimizer_key_uses": [
              {
                "table": "`item_sale_summary`",
                "field": "ent_id",
                "equals": "1747964630024192400",
                "null_rejecting": false
              },
              {
                "table": "`item_sale_summary`",
                "field": "ent_id",
                "equals": "1747964630024192400",
                "null_rejecting": false
              }
            ]
          },
          {
            "rows_estimation": [  # 预估不同单表访问方法的访问成本
              {
                "table": "`item_sale_summary`",
                "range_analysis": {
                  "table_scan": { # 全表扫描的行数以及成本
                    "rows": 4245934,
                    "cost": 944293
                  },
                  "potential_range_indexes": [  # 分析可能使用的索引
                    {
                      "index": "unique_index",
                      "usable": true, # 可能被使用
                      "key_parts": [
                        "trans_date",
                        "ent_id",
                        "region_code",
                        "channel_keyword",
                        "item_code",
                        "barcode"
                      ]
                    },
                    {
                      "index": "idx_ent_date_region",
                      "usable": true,
                      "key_parts": [
                        "ent_id",
                        "trans_date",
                        "region_code"
                      ]
                    },
                    {
                      "index": "idx_saled_item",
                      "usable": true,
                      "key_parts": [
                        "ent_id",
                        "region_code",
                        "item_code",
                        "barcode",
                        "trans_date",
                        "channel_keyword"
                      ]
                    }
                  ],
                  "best_covering_index_scan": {
                    "index": "unique_index",
                    "cost": 1.11e6,
                    "chosen": false,
                    "cause": "cost"
                  },
                  "setup_range_conditions": [
                  ],
                  "group_index_range": {
                    "chosen": false,
                    "cause": "not_applicable_aggregate_function"
                  },
                  "analyzing_range_alternatives": {  # 分析各种可能使用的索引的成本
                    "range_scan_alternatives": [
                      {
                        # 使用unique_index的成本分析
                        "index": "unique_index",
                        "ranges": [
                          "0x61cb0f <= trans_date <= 0x6fcb0f"
                        ],
                        "index_dives_for_eq_ranges": true, # 是否使用index dive
                        "rowid_ordered": false, # 使用该索引获取的记录是否按照主键排序
                        "using_mrr": false, # 是否使用mrr
                        "index_only": true, # 是否是索引覆盖访问
                        "rows": 1, # 使用该索引获取的记录条数
                        "cost": 1.21, # 使用该索引的成本
                        "chosen": true # 是否选择该索引
                      },
                      {
                        "index": "idx_ent_date_region",
                        "ranges": [
                          "1747964630024192400 <= ent_id <= 1747964630024192400 AND 0x61cb0f <= trans_date <= 0x6fcb0f"
                        ],
                        "index_dives_for_eq_ranges": true,
                        "rowid_ordered": false,
                        "using_mrr": false,
                        "index_only": false,
                        "rows": 1,
                        "cost": 2.21,
                        "chosen": false,
                        "cause": "cost" # 因为成本太大所以不选择该索引
                      },
                      {
                        "index": "idx_saled_item",
                        "ranges": [
                          "1747964630024192400 <= ent_id <= 1747964630024192400"
                        ],
                        "index_dives_for_eq_ranges": true,
                        "rowid_ordered": false,
                        "using_mrr": false,
                        "index_only": true,
                        "rows": 753872,
                        "cost": 197892,
                        "chosen": false,
                        "cause": "cost"
                      }
                    ],
                    "analyzing_roworder_intersect": {
                      # 分析使用索引合并的成本
                      "usable": false,
                      "cause": "too_few_roworder_scans"
                    }
                  },
                  "chosen_range_access_summary": {
                    # 对于上述单表查询最优的访问方法
                    "range_access_plan": {
                      "type": "range_scan",
                      "index": "unique_index",
                      "rows": 1,
                      "ranges": [
                        "0x61cb0f <= trans_date <= 0x6fcb0f"
                      ]
                    },
                    "rows_for_plan": 1,
                    "cost_for_plan": 1.21,
                    "chosen": true
                  }
                }
              }
            ]
          },
          {
            # 分析各种可能的执行计划
            "considered_execution_plans": [
              {
                "plan_prefix": [
                ],
                "table": "`item_sale_summary`",
                "best_access_path": {
                  "considered_access_paths": [
                    {
                      "access_type": "ref",
                      "index": "idx_ent_date_region",
                      "rows": 710.55,
                      "cost": 852.66,
                      "chosen": true
                    },
                    {
                      "access_type": "ref",
                      "index": "idx_saled_item",
                      "rows": 753872,
                      "cost": 197892,
                      "chosen": false
                    },
                    {
                      "rows_to_scan": 1,
                      "access_type": "range",
                      "range_details": {
                        "used_index": "unique_index"
                      },
                      "resulting_rows": 0.1776,
                      "cost": 1.41,
                      "chosen": true,
                      "use_tmp_table": true
                    }
                  ]
                },
                "condition_filtering_pct": 100,
                "rows_for_plan": 0.1776,
                "cost_for_plan": 1.41,
                "sort_cost": 0.1776,
                "new_cost_for_plan": 1.5876,
                "chosen": true
              }
            ]
          },
          {
            # 尝试给查询添加一些其他的查询条件
            "attaching_conditions_to_tables": {
              "original_condition": "((`item_sale_summary`.`ent_id` = 1747964630024192400) and (`item_sale_summary`.`trans_date` between '2021-11-01' and '2021-11-15'))",
              "attached_conditions_computation": [
              ],
              "attached_conditions_summary": [
                {
                  "table": "`item_sale_summary`",
                  "attached": "((`item_sale_summary`.`ent_id` = 1747964630024192400) and (`item_sale_summary`.`trans_date` between '2021-11-01' and '2021-11-15'))"
                }
              ]
            }
          },
          {
            "clause_processing": {
              "clause": "GROUP BY",
              "original_clause": "`item_sale_summary`.`ent_id`,`item_sale_summary`.`region_code`",
              "items": [
                {
                  "item": "`item_sale_summary`.`ent_id`",
                  "equals_constant_in_where": true
                },
                {
                  "item": "`item_sale_summary`.`region_code`"
                }
              ],
              "resulting_clause_is_simple": true,
              "resulting_clause": "`item_sale_summary`.`region_code`"
            }
          },
          {
            "reconsidering_access_paths_for_index_ordering": {
              "clause": "GROUP BY",
              "steps": [
              ],
              "index_order_summary": {
                "table": "`item_sale_summary`",
                "index_provides_order": false,
                "order_direction": "undefined",
                "index": "unique_index",
                "plan_changed": false
              }
            }
          },
          {
            # 再稍稍的改进一下执行计划
            "refine_plan": [
              {
                "table": "`item_sale_summary`"
              }
            ]
          },
          {
            "creating_tmp_table": {
              "tmp_table_info": {
                "table": "intermediate_tmp_table",
                "row_length": 344,
                "key_length": 349,
                "unique_constraint": false,
                "location": "memory (heap)",
                "row_limit_estimate": 48770
              }
            }
          }
        ]
      }
    },
    {
      # execute阶段
      "join_execution": {
        "select#": 1,
        "steps": [
          {
            "filesort_information": [
              {
                "direction": "asc",
                "table": "`item_sale_summary`",
                "field": "region_code"
              }
            ],
            "filesort_priority_queue_optimization": {
              "usable": false,
              "cause": "not applicable (no LIMIT)"
            },
            "filesort_execution": [
            ],
            "filesort_summary": {
              "rows": 0,
              "examined_rows": 0,
              "number_of_tmp_files": 0,
              "sort_buffer_size": 261632,
              "sort_mode": "<sort_key, packed_additional_fields>"
            }
          }
        ]
      }
    }
  ]
}
```

想要更加具体的了解其中的含义可参考[Chapter 8 Tracing the Optimizer](https://dev.mysql.com/doc/internals/en/optimizer-tracing.html)

## 4. 总结

以上为`optimizer trace`的简单使用，使用好该功能可以有效帮助我们了解MySQL的优化过程。

整体优化过程虽然看起来杂乱，但主要分成了以下三个部分

- prepare 阶段
- optimize 阶段
- execute 阶段

的基于成本的优化主要集中在 optimize 阶段，对于单表查询来说，我们主要关注 optimize 阶段的 "rows_estimation" 这个过程，这个过程深入分析了对单表查询的各种执行方案的成本；对于多表连接查询来说，我们更多需要关注 "considered_execution_plans" 这个过程，这个过程里会写明各种不同的连接方式所对应的成本。反正优化器最终会选择成本最低的那种方案来作为最终的执行计划，也就是我们使用 EXPLAIN 语句所展现出的那种方案。

## 5. 参考

- *《MySQL是怎样运行的》*
- *[MySQL · 最佳实践 · 性能分析的大杀器—Optimizer trace](http://mysql.taobao.org/monthly/2019/11/03/)*
- *[Chapter 8 Tracing the Optimizer](https://dev.mysql.com/doc/internals/en/optimizer-tracing.html)*