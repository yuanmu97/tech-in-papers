# 12-conditional functional dependency
Source: Sampling from repairs of conditional functional dependency violations, VLDB 2014

keywords: data cleaning, conditional functional dependency, integrity constraint violation

解决 functional dependencies & conditional FDs 的一个挑战在于 exponential number of possible repairs，本文提出不只局限于找到一个 single repair，而是在可能的 repairs 空间里做 sampling 的想法



business data 的 quality 受到各种 noise sources 的损害，例如数据格式的异质性、信息提取器的不完善和数据生成设备的不精确。

* 这一点对我们的业务场景是准确的。噪声主要来自于员工的操作不精确，实际上就是“数据生成设备的不精确”。



FDs 可以被视为一种编码了数据语义的 integrity constraints；在实践中，往往在集成了多个异构数据后或者从 Web 上抽取数据之后，会发生 FDs violations



CFDs 是对 FDs 的一个扩展，一个 CFD 由一个模板 FD 和包含一组模式的 Tableau 组成。

​	例如：`(State, ZIP → City, (‘MI’, _, _))` 表示对于 `State, ZIP -> City` 这个 FD 而言，仅对于那些 `State="MI"` 的元组成立。简单来说就是有条件的 FDs



在做 repair 时，有三种方式：

1. single, nearly optimal repair，一般考虑的是最少次数的修改
2. consistent query answering，针对一个特定的 query，计算对每个合理 repairs 中都有效的 answer
3. domain expert 人工 clean data

但在一些应用下，单个 repair 或者 consistent query answering 都是不够的。

本文重在设计一种能够高效地生成出足够有意义的 repairs samples



**结论**

* functional dependency 以及其扩展 conditional FD 能够表达一部分我们关心的约束，例如：
  * task_id -> task_type, task priority
  * employee_id -> department_id, position
* 但 FD、CFD 表达的是 attributes 之间的依赖关系，无法描述多个 records 之间的约束，例如：
  * 如果 task_id_1 是 task_id_2 的前置任务，则 task_id_1 的 finish_date 必须早于 task_id_2 的 start_date

