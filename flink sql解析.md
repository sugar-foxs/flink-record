#  Flink sql 解析

以查询为例，sql解析从TableEnvironment的sqlQuery方法开始。sql语句作为参数，过程如下。

```
def sqlQuery(query: String): Table = {
    val planner = new FlinkPlannerImpl(getFrameworkConfig, getPlanner, getTypeFactory)
    // 将sql语句变成sqlNode Tree
    val parsed = planner.parse(query)
    if (null != parsed && parsed.getKind.belongsTo(SqlKind.QUERY)) {
      // 结合catalog验证sql语法
      val validated = planner.validate(parsed)
      // 生成Calcite逻辑计划
      val relational = planner.rel(validated)
      new Table(this, LogicalRelNode(relational.rel))
    } else {
      throw new TableException(
        "Unsupported SQL query! sqlQuery() only accepts SQL queries of type " +
          "SELECT, UNION, INTERSECT, EXCEPT, VALUES, and ORDER_BY.")
    }
  }
```



- parse(sql)，基于Java CC的Parse.jj文件，将sql语句变成sqlNode Tree。javacc的解析过程之后详细分析，现在只需要知道sql语句被解析成sqlNode构成的树。
- validate(sqlNode)，结合catalog验证sql语法。
- rel(sqlNode)，生成Calcite逻辑计划（RelNode tree）。
- 基于优化规则区优化逻辑树，生成flink的物理执行计划physical Plan。
- 物理执行计划转成Flink可执行的计划。

先对后面可能会遇到的Calcite中名词进行解释一下，

- SqlNode，sqlTree中的节点，将在SqlToRelConverter中转化为RelNode。
- RexNode，row表达式，比如RexLiteral是常量表达式，如”123”;RexCall是函数表达式，如cast(xx as xx)。它们都是RexNode的子类。
- RelNode，关系表达式，用来处理数据，如Sort, Join, Project, Filter, Scan, Sample。
- RelTrait，特征，RelNode中有特征集合，比如RelCollation可能是Project中的排序特征。
- RelTraitDef，特征定义，定义了Trait对应的一些方法。
- RelSubset，表示带有同一Trait的RelNode集合。
- Convention，转化特征，继承自RelTrait，用于转化RelNode，常见的有FlinkConvention
- Literal，常量
- Planner，SQL计划，用于解析、优化、执行

## 1，生成语法树

生成语法树是使用Calcite的SqlParse，这里着重讲flink，涉及到calcite就先不介绍了，只需知道sql字符串被转化成了sqlNode tree。

## 2，验证sql语法

validate是使用FlinkCalciteSqlValidator去验证sql语法，FlinkCalciteValidator继承自Calcite中验证语法的默认实现SqlValidatorImpl类。往下追溯到核心逻辑performUnconditionalRewrites方法，这是将表达式重写成标准形式的方法，以便让验证逻辑的其余部分更加简单。

使用深度遍历将tree中每一个sqlNode都转变成标准形式。下面是对每一种类型的sqlNode进行transform的方法。

```
// now transform node itself
    final SqlKind kind = node.getKind();
    switch (kind) {
    case VALUES:
      // CHECKSTYLE: IGNORE 1
      if (underFrom || true) {
        // leave FROM (VALUES(...)) [ AS alias ] clauses alone,
        // otherwise they grow cancerously if this rewrite is invoked
        // over and over
        return node;
      } else {
        final SqlNodeList selectList =
            new SqlNodeList(SqlParserPos.ZERO);
        selectList.add(SqlIdentifier.star(SqlParserPos.ZERO));
        return new SqlSelect(node.getParserPosition(), null, selectList, node,
            null, null, null, null, null, null, null);
      }

    case ORDER_BY: {
      SqlOrderBy orderBy = (SqlOrderBy) node;
      handleOffsetFetch(orderBy.offset, orderBy.fetch);
      if (orderBy.query instanceof SqlSelect) {
        SqlSelect select = (SqlSelect) orderBy.query;

        // Don't clobber existing ORDER BY.  It may be needed for
        // an order-sensitive function like RANK.
        if (select.getOrderList() == null) {
          // push ORDER BY into existing select
          select.setOrderBy(orderBy.orderList);
          select.setOffset(orderBy.offset);
          select.setFetch(orderBy.fetch);
          return select;
        }
      }
      if (orderBy.query instanceof SqlWith
          && ((SqlWith) orderBy.query).body instanceof SqlSelect) {
        SqlWith with = (SqlWith) orderBy.query;
        SqlSelect select = (SqlSelect) with.body;

        // Don't clobber existing ORDER BY.  It may be needed for
        // an order-sensitive function like RANK.
        if (select.getOrderList() == null) {
          // push ORDER BY into existing select
          select.setOrderBy(orderBy.orderList);
          select.setOffset(orderBy.offset);
          select.setFetch(orderBy.fetch);
          return with;
        }
      }
      final SqlNodeList selectList = new SqlNodeList(SqlParserPos.ZERO);
      selectList.add(SqlIdentifier.star(SqlParserPos.ZERO));
      final SqlNodeList orderList;
      if (getInnerSelect(node) != null && isAggregate(getInnerSelect(node))) {
        orderList = SqlNode.clone(orderBy.orderList);
        // We assume that ORDER BY item does not have ASC etc.
        // We assume that ORDER BY item is present in SELECT list.
        for (int i = 0; i < orderList.size(); i++) {
          SqlNode sqlNode = orderList.get(i);
          SqlNodeList selectList2 = getInnerSelect(node).getSelectList();
          for (Ord<SqlNode> sel : Ord.zip(selectList2)) {
            if (stripAs(sel.e).equalsDeep(sqlNode, Litmus.IGNORE)) {
              orderList.set(i,
                  SqlLiteral.createExactNumeric(Integer.toString(sel.i + 1),
                      SqlParserPos.ZERO));
            }
          }
        }
      } else {
        orderList = orderBy.orderList;
      }
      return new SqlSelect(SqlParserPos.ZERO, null, selectList, orderBy.query,
          null, null, null, null, orderList, orderBy.offset,
          orderBy.fetch);
    }

    case EXPLICIT_TABLE: {
      // (TABLE t) is equivalent to (SELECT * FROM t)
      SqlCall call = (SqlCall) node;
      final SqlNodeList selectList = new SqlNodeList(SqlParserPos.ZERO);
      selectList.add(SqlIdentifier.star(SqlParserPos.ZERO));
      return new SqlSelect(SqlParserPos.ZERO, null, selectList, call.operand(0),
          null, null, null, null, null, null, null);
    }

    case DELETE: {
      SqlDelete call = (SqlDelete) node;
      SqlSelect select = createSourceSelectForDelete(call);
      call.setSourceSelect(select);
      break;
    }

    case UPDATE: {
      SqlUpdate call = (SqlUpdate) node;
      SqlSelect select = createSourceSelectForUpdate(call);
      call.setSourceSelect(select);

      // See if we're supposed to rewrite UPDATE to MERGE
      // (unless this is the UPDATE clause of a MERGE,
      // in which case leave it alone).
      if (!validatingSqlMerge) {
        SqlNode selfJoinSrcExpr =
            getSelfJoinExprForUpdate(
                call.getTargetTable(),
                UPDATE_SRC_ALIAS);
        if (selfJoinSrcExpr != null) {
          node = rewriteUpdateToMerge(call, selfJoinSrcExpr);
        }
      }
      break;
    }

    case MERGE: {
      SqlMerge call = (SqlMerge) node;
      rewriteMerge(call);
      break;
    }
    }
```

转变之后，调用每个SqlNode的validate方法，都是calcite内部的方法，以SqlSelect为例，最后调用的是validateQuery方法：

```
public void validateQuery(SqlNode node, SqlValidatorScope scope,
      RelDataType targetRowType) {
    final SqlValidatorNamespace ns = getNamespace(node, scope);
    if (node.getKind() == SqlKind.TABLESAMPLE) {
      List<SqlNode> operands = ((SqlCall) node).getOperandList();
      SqlSampleSpec sampleSpec = SqlLiteral.sampleValue(operands.get(1));
      if (sampleSpec instanceof SqlSampleSpec.SqlTableSampleSpec) {
        validateFeature(RESOURCE.sQLFeature_T613(), node.getParserPosition());
      } else if (sampleSpec
          instanceof SqlSampleSpec.SqlSubstitutionSampleSpec) {
        validateFeature(RESOURCE.sQLFeatureExt_T613_Substitution(),
            node.getParserPosition());
      }
    }

    validateNamespace(ns, targetRowType);
    if (node == top) {
      validateModality(node);
    }
    validateAccess(
        node,
        ns.getTable(),
        SqlAccessEnum.SELECT);
  }
```



## 3，生成Calcite的逻辑计划

```
val relational = planner.rel(validated)
```

内部使用SqlToRelConverter的convertQuery方法将验证过的sqlNode转变为RelRoot。

```
public RelRoot convertQuery(
      SqlNode query,
      final boolean needsValidation,
      final boolean top) {
    //前面如果没有进行验证，在这可以进行验证
    if (needsValidation) {
      query = validator.validate(query);
    }

    RelMetadataQuery.THREAD_PROVIDERS.set(
    JaninoRelMetadataProvider.of(cluster.getMetadataProvider()));
    //递归进行转变成RelNode,真正生成逻辑计划的过程在这个方法里面
    RelNode result = convertQueryRecursive(query, top, null).rel;
    if (top) {
      if (isStream(query)) {
        //节点是根节点且是关于流的查询
        result = new LogicalDelta(cluster, result.getTraitSet(), result);
      }
    }
    //RelCollation表示表中列排序顺序和方向
    RelCollation collation = RelCollations.EMPTY;
    if (!query.isA(SqlKind.DML)) {
      if (isOrdered(query)) {
        collation = requiredCollation(result);
      }
    }
    //检查查询的结果类型是否正确
    checkConvertedType(query, result);

    if (SQL2REL_LOGGER.isDebugEnabled()) {
      SQL2REL_LOGGER.debug(
          RelOptUtil.dumpPlan("Plan after converting SqlNode to RelNode",
              result, SqlExplainFormat.TEXT,
              SqlExplainLevel.EXPPLAN_ATTRIBUTES));
    }
    //获取所有field类型
    final RelDataType validatedRowType = validator.getValidatedNodeType(query);
    return RelRoot.of(result, validatedRowType, query.getKind())
        .withCollation(collation);
  }
```



## 4，优化逻辑计划，生成flink逻辑计划

优化逻辑在writeToSink方法中引用，使用的RBO优化方式，即事先定义一系列的规则，然后根据这些规则来优化执行计划。

```
private[flink] def optimize(relNode: RelNode, updatesAsRetraction: Boolean): RelNode = {

    // 0. 在查询去相关之前转换子查询，使用的是TABLE_SUBQUERY_RULES规则
    val convSubQueryPlan = runHepPlanner(
      HepMatchOrder.BOTTOM_UP, FlinkRuleSets.TABLE_SUBQUERY_RULES, relNode, relNode.getTraitSet)
    LOG.debug(s"----------convSubQueryPlan--------------\n${RelOptUtil.toString(convSubQueryPlan)}")

    // 0. 使用TABLE_REF_RULES规则，在查询去相关之前转换表引用
    val fullRelNode = runHepPlanner(
      HepMatchOrder.BOTTOM_UP,
      FlinkRuleSets.TABLE_REF_RULES,
      convSubQueryPlan,
      relNode.getTraitSet)
    LOG.debug(s"----------fullRelNode--------------\n${RelOptUtil.toString(fullRelNode)}")

    // 1. 去除关联子查询
    val decorPlan = RelDecorrelator.decorrelateQuery(fullRelNode)
    LOG.debug(s"----------decorPlan--------------\n${RelOptUtil.toString(decorPlan)}")

    // 2. 转换time的标识符，比如存在rowtime标识的话，我们将会引入TimeMaterializationSqlFunction operator
    val convPlan = RelTimeIndicatorConverter.convert(decorPlan, getRelBuilder.getRexBuilder)
    LOG.debug(s"----------convPlan--------------\n${RelOptUtil.toString(convPlan)}")

    // 3. 规范化logic计划 , 比如一个Filter的过滤条件都是true的话，我们可以直接将这个filter去掉
    //默认使用的是DATASTREAM_NORM_RULES或DATASET_NORM_RULES规则
    val normRuleSet = getNormRuleSet
    val normalizedPlan = if (normRuleSet.iterator().hasNext) {
      runHepPlanner(HepMatchOrder.BOTTOM_UP, normRuleSet, convPlan, convPlan.getTraitSet)
    } else {
      convPlan
    }
    LOG.debug(s"----------normalizedPlan--------------\n${RelOptUtil.toString(normalizedPlan)}")

    // 4. 优化逻辑计划，调整节点间的上下游到达优化计算逻辑的效果，同时将节点转换成派生于FlinkLogicalRel的节点
    //默认使用的是LOGICAL_OPT_RULES规则
    val logicalOptRuleSet = getLogicalOptRuleSet
    val logicalOutputProps = relNode.getTraitSet.replace(FlinkConventions.LOGICAL).simplify()
    val logicalPlan = if (logicalOptRuleSet.iterator().hasNext) {
      runVolcanoPlanner(logicalOptRuleSet, normalizedPlan, logicalOutputProps)
    } else {
      normalizedPlan
    }
    LOG.debug(s"----------logicalPlan--------------\n${RelOptUtil.toString(logicalPlan)}")

    // 5. 将优化后的逻辑计划转换成Flink的物理计划，同时将节点转换成派生于DataStreamRel的节点
    //默认使用的是DATASTREAM_OPT_RULES或DATASET_OPT_RULES规则
    val physicalOptRuleSet = getPhysicalOptRuleSet
    val physicalOutputProps = relNode.getTraitSet.replace(FlinkConventions.DATASTREAM).simplify()
    val physicalPlan = if (physicalOptRuleSet.iterator().hasNext) {
      runVolcanoPlanner(physicalOptRuleSet, logicalPlan, physicalOutputProps)
    } else {
      logicalPlan
    }
    LOG.debug(s"----------physicalPlan--------------\n${RelOptUtil.toString(physicalPlan)}")

    // 6. 装饰物理计划，默认使用的是DATASTREAM_DECO_RULES规则
    val decoRuleSet = getDecoRuleSet
    val decoratedPlan = if (decoRuleSet.iterator().hasNext) {
      val planToDecorate = if (updatesAsRetraction) {
        physicalPlan.copy(
          physicalPlan.getTraitSet.plus(new UpdateAsRetractionTrait(true)),
          physicalPlan.getInputs)
      } else {
        physicalPlan
      }
      runHepPlanner(
        HepMatchOrder.BOTTOM_UP,
        decoRuleSet,
        planToDecorate,
        planToDecorate.getTraitSet)
    } else {
      physicalPlan
    }
    LOG.debug(s"----------decoratedPlan--------------\n${RelOptUtil.toString(decoratedPlan)}")

    decoratedPlan
  }
```

optimize方法其实是使用各种提前定制好的Rule对计划进行优化，具体的Rule逻辑在这里先不讲了，之后具体介绍，用户也可通过继承RelOptRule类定义自己的Rule。

## 5，生成flink可执行计划

translate()在TableEnvironment.writeToSink中被调用,主要逻辑是将优化后的物理计划(RelNode,流处理是DataStreamRel,批处理是DataSetRel)转变为flink可执行的计划(DataStream)。

```
protected def translate[A](
      logicalPlan: RelNode,
      logicalType: RelDataType,
      queryConfig: StreamQueryConfig,
      withChangeFlag: Boolean)
      (implicit tpe: TypeInformation[A]): DataStream[A] = {

    // if no change flags are requested, verify table is an insert-only (append-only) table.
    if (!withChangeFlag && !UpdatingPlanChecker.isAppendOnly(logicalPlan)) {
      throw new TableException(
        "Table is not an append-only table. " +
        "Use the toRetractStream() in order to handle add and retract messages.")
    }

    // 这个方法是转换核心，其实是调用各个DataStreamRel的tranlateToPlan
    val plan: DataStream[CRow] = translateToCRow(logicalPlan, queryConfig)

    val rowtimeFields = logicalType
      .getFieldList.asScala
      .filter(f => FlinkTypeFactory.isRowtimeIndicatorType(f.getType))

    // convert the input type for the conversion mapper
    // the input will be changed in the OutputRowtimeProcessFunction later
    val convType = if (rowtimeFields.size > 1) {
      throw new TableException(
        s"Found more than one rowtime field: [${rowtimeFields.map(_.getName).mkString(", ")}] in " +
          s"the table that should be converted to a DataStream.\n" +
          s"Please select the rowtime field that should be used as event-time timestamp for the " +
          s"DataStream by casting all other fields to TIMESTAMP.")
    } else if (rowtimeFields.size == 1) {
      val origRowType = plan.getType.asInstanceOf[CRowTypeInfo].rowType
      val convFieldTypes = origRowType.getFieldTypes.map { t =>
        if (FlinkTypeFactory.isRowtimeIndicatorType(t)) {
          SqlTimeTypeInfo.TIMESTAMP
        } else {
          t
        }
      }
      CRowTypeInfo(new RowTypeInfo(convFieldTypes, origRowType.getFieldNames))
    } else {
      plan.getType
    }

    // 转换CRow至输出类型的MapFunction
    val conversion: MapFunction[CRow, A] = if (withChangeFlag) {
      getConversionMapperWithChanges(
        convType,
        new RowSchema(logicalType),
        tpe,
        "DataStreamSinkConversion")
    } else {
      getConversionMapper(
        convType,
        new RowSchema(logicalType),
        tpe,
        "DataStreamSinkConversion")
    }

    val rootParallelism = plan.getParallelism
    //执行上面的MapFunction,设置rowtime
    val withRowtime = if (rowtimeFields.isEmpty) {
      plan.map(conversion)
    } else {
      plan.process(new OutputRowtimeProcessFunction[A](conversion, rowtimeFields.head.getIndex))
    }

    withRowtime
      .returns(tpe)
      .name(s"to: ${tpe.getTypeClass.getSimpleName}")
      .setParallelism(rootParallelism)
  }
```

translateToCRow方法调用各个DataStreamRel的translateToPlan方法，利用CodeGen元编程成Flink的各种算子。下面以比较简单的DataStreamValues为例介绍它的translateToPlan方法：

```
override def translateToPlan(
      tableEnv: StreamTableEnvironment,
      queryConfig: StreamQueryConfig): DataStream[CRow] = {

    val config = tableEnv.getConfig

    val returnType = CRowTypeInfo(schema.typeInfo)
    val generator = new InputFormatCodeGenerator(config)

    // 为每条记录生成代码
    val generatedRecords = getTuples.asScala.map { r =>
      generator.generateResultExpression(
        schema.typeInfo,
        schema.fieldNames,
        r.asScala)
    }

    // 生成input format
    val generatedFunction = generator.generateValuesInputFormat(
      ruleDescription,
      generatedRecords.map(_.code),
      schema.typeInfo)

    val inputFormat = new CRowValuesInputFormat(
      generatedFunction.name,
      generatedFunction.code,
      returnType)
    //产生DataStreamSource
    tableEnv.execEnv.createInput(inputFormat, returnType)
  }
```











