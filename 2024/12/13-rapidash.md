# 13-rapidash
Source: Rapidash: Efficient Detection of Constraint Violations, VLDB 2024

拒绝约束（Denial Constraint）能捕捉各种常见的完整性约束，包括候选键（candidate keys）、FD 和排序约束（ordering constraints）等。

本文重点关注的也是大规模 DB 场景下的开销问题，当前 SOTA 的时空复杂度对于 rows# 都是二次的。



Integrity constraints：根据 gpt 的回复，完整性约束主要包括以下几种类型：

1. **实体完整性（Entity Integrity）**：
   - 确保每个表中的每一行都有唯一的标识符（主键），且主键值不能为空。
2. **参照完整性（Referential Integrity）**：
   - 确保外键值必须在主表的主键中存在。即，如果一个表中的字段引用了另一个表的字段，那么引用的值必须有效。
3. **域完整性（Domain Integrity）**：
   - 确保字段中的数据值属于特定的域（数据类型、范围或格式），例如，年龄字段只能包含非负整数。
4. **用户定义的完整性（User-defined Integrity）**：
   - 根据特定业务规则定义的约束，比如某个字段的值必须在特定的范围内。

本文则将 denial constraint, DC 也作为一类 Integrity constraint，描述为能够捕捉很多有用的 Integrity constraints，包括 FD，ordering constraints，unique column combinations 等，以下表为例：

![image-20241213150142859](/Users/yuanmu/Desktop/tech-in-papers/2024/12/image-20241213150142859.png)

文章介绍了 DC 可以表示的三种 constraints：

* SSN 是一个 candidate key：成立。
  * 形式化为 $\varphi_1: \neg (s.SSN=t.SSN)$
* Zip -> State 是一个 FD：成立。
  * 形式化为 $\varphi_2: \neg (s.Zip = t.Zip \and s.State = t.State)$
* 对于在相同 State 的人，如果 A 的 Salary 不小于 B，则 A 的 FedTaxRate 也不小于 B：存在 violations，因为 t2 t3 的 salary 都比 t4 更高，但 fedtaxrate 却更低。
  * 形式化为 $\varphi_3: \neg (s.State=t.State \and s.Salary \leq t.Salary \and s.FedTaxRate > t.FedTaxRate)$

更 general 来说，DC 表示如下形式：
$$
\forall s,t \in R, s\neq t: \neg(p_1 \and \cdots \and p_m)
$$
其中 $R$ 表示一个 relation，而 $s,t$ 为 tuples， 每个 $p_i$ 表示为 s 和 t 的属性之间的关系，包括 $=,\neq,>,<,\geq, \leq$ 六种。



检测 DC 的过程是在 DB 上验证 DC 是否满足，这一步称为 DC verification；检测出违反 DC 的结果用一个 tuple 进行表示，例如 (t2, t4) 和 (t3, t4) 违反了上面例子的第三个 DC；之后，这些 tuples 可以交给 data cleaning tools 进行修复。



本文提出利用 orthogonal range search 方法进行检测，仍以上面的例子，可以将 t2, t3, t4 在一个二维坐标系里表示为点，第三个 DC 则可以表示为一个如下图的红色阴影区域：

![image-20241213161825515](/Users/yuanmu/Desktop/tech-in-papers/2024/12/image-20241213161825515.png)

但是，通过 vanilla 的搜索算法，性能没有提升，本文提出了针对性的优化，显著降低复杂度。



**结论**

* denial constraints 能够表达两行之间的限制，符合一些我们场景的规则，例如规则“如果 task_id_1 是 task_id_2 的前置任务，则 task_id_1 的 finish_date 必须早于 task_id_2 的 start_date”可以表示为： $\neg (s.ChildTaskId=t.TaskId \and s.FinishDate > t.StartDate)$​
* 但是更为复杂的，比如关于多于两个 rows 之间的关系限制，理论上是不能表示的



关于两 rows 以上的 DCs，GPT4 回复如下：

Denial constraints (DCs) are indeed a powerful tool for expressing data dependencies and ensuring data integrity by specifying rules that prohibit certain combinations of tuples in a database. While DCs are traditionally expressed over pairs of tuples, they can be extended to work over three or more tuples to capture more complex relationships. Here's how this can be achieved, along with some alternative techniques you can explore:

---

### **1. Extending Denial Constraints for Multiple Tuples**
To extend DCs over three or more tuples, the general approach is to write Boolean quantifiers and predicates that involve additional variables corresponding to the tuples. A denial constraint over three tuples can be written as:

```plaintext
∀t1, t2, t3 ∈ R: ¬(P(t1, t2, t3))
```

Where `P(t1, t2, t3)` is a Boolean predicate that defines the prohibited relationship between the three tuples `t1`, `t2`, and `t3`. The predicates can include comparisons (e.g., equality, inequality) and logical conditions involving attributes of the tuples. For example:

#### Example:【这个例子挺符合】
Suppose you have a table `Orders(OrderID, CustomerID, TotalAmount)` and want to enforce the rule: "No three orders from the same customer can have the same total amount." The corresponding denial constraint would be:

```plaintext
∀t1, t2, t3 ∈ Orders: ¬(t1.CustomerID = t2.CustomerID ∧ t2.CustomerID = t3.CustomerID ∧ t1.TotalAmount = t2.TotalAmount ∧ t2.TotalAmount = t3.TotalAmount)
```

This ensures that no such three tuples can coexist in the `Orders` relation.

#### Implementation:
- **Declarative Frameworks**: If your database system supports declarative constraint frameworks, you can encode such DCs directly in the query language.
- **Triggers**: In systems without declarative DC support, you can implement these constraints using triggers that check the conditions whenever a new tuple is inserted or updated.

---

### **2. Other Techniques for Extending DCs**
When handling complex multi-tuple constraints, here are some additional techniques you can consider:

#### **a. Using Aggregate Functions**
For constraints that involve more than two tuples, aggregate functions can be used to express conditions over groups of tuples. For example, SQL constraints or queries can be written to check for violations using `GROUP BY` and `HAVING`.

#### Example:
In the "three orders with the same total amount" scenario, you could express this as:

```sql
SELECT CustomerID, TotalAmount
FROM Orders
GROUP BY CustomerID, TotalAmount
HAVING COUNT(*) >= 3;
```

If this query returns any result, it indicates a violation of the constraint.

---

#### **b. Multi-Tuple Triggers**
Database triggers can be extended to check conditions over multiple tuples. For example, a trigger can fire on `INSERT` or `UPDATE` and examine all relevant tuples to detect violations.

#### Example (Pseudo-code):
```sql
CREATE TRIGGER CheckMultiTupleConstraint
AFTER INSERT OR UPDATE ON Orders
FOR EACH ROW
BEGIN
    DECLARE violation_count INT;
    SELECT COUNT(*)
    INTO violation_count
    FROM Orders
    WHERE CustomerID = NEW.CustomerID AND TotalAmount = NEW.TotalAmount;
    IF violation_count >= 3 THEN
        SIGNAL SQLSTATE '45000' SET MESSAGE_TEXT = 'Constraint violation: More than two orders have the same total amount for this customer.';
    END IF;
END;
```

This trigger ensures that no insertion or update violates the multi-tuple constraint.

---

#### **c. Tuple-Generating Dependencies (TGDs)**【我觉得这个挺有意思，可以再去调研下】
Tuple-generating dependencies are a more general form of constraints in logic-based frameworks. TGDs allow you to express relationships between tuples in a more declarative manner, often using Datalog or first-order logic-like syntax.

For example, you could write:

```plaintext
∀t1, t2, t3 ∈ R: (t1.CustomerID = t2.CustomerID ∧ t2.CustomerID = t3.CustomerID ∧ t1.TotalAmount = t2.TotalAmount ∧ t2.TotalAmount = t3.TotalAmount) → ⊥
```

This rule states that whenever the left-hand side is true (three tuples with the same `CustomerID` and `TotalAmount`), a contradiction (`⊥`) arises, effectively disallowing such combinations.

---

#### **d. Logic Programming and Datalog**
For more complex and extensible scenarios, logic programming languages like Prolog or Datalog can be used. These languages allow you to define constraints declaratively and reason about multi-tuple relationships easily.

#### Example in Datalog:
```datalog
violation(CustomerID) :-
    Orders(t1, CustomerID, TotalAmount),
    Orders(t2, CustomerID, TotalAmount),
    Orders(t3, CustomerID, TotalAmount),
    t1 != t2, t2 != t3, t1 != t3.
```

This rule identifies violations where three distinct tuples have the same `CustomerID` and `TotalAmount`.

---

### **3. Alternative Techniques for Complex Constraints**
In addition to extending DCs, you might find the following useful:

#### **a. Temporal Constraints**
If the constraint extends over time (e.g., no three orders in the same hour), you can use temporal constraints and query languages that support time-based reasoning, such as TSQL or temporal logic systems.

#### **b. Graph-Based Approaches**
For constraints that involve relationships among many tuples (e.g., transitive constraints in a social network), modeling the data as a graph and using graph algorithms or databases like Neo4j can help.

#### **c. Machine Learning for Anomaly Detection**
If the constraints are too complex to specify explicitly, you can use machine learning models to detect anomalies or patterns that suggest a violation of implicit constraints.

---

### **4. Trade-Offs**
While extending DCs or using alternative techniques, keep in mind:
- **Performance**: Checking constraints over multiple tuples can be computationally expensive, especially for large datasets.
- **Expressiveness vs. Complexity**: More expressive constraints are often harder to implement and maintain.
- **Scalability**: Consider how the chosen method scales with increasing data volume.

By combining these techniques, you can model and enforce denial constraints over three or more tuples effectively.
