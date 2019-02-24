使用javacc和Freemarker 扩展模板Parser.jj,将自己的语法加入到其中。

- Freemarker负责将增加的内容加入到模板中

config.fmpp文件是FMPP的配置文件
```
data: {
  parser: tdd(../data/Parser.tdd)
}

freemarkerLinks: {
  includes: includes/
}
```
Parser.tdd中的内容是将插入到Parser.jj中的内容，比如classname，package，imports，还有方法名等。

在Parser.jj文件中有
```
<#list parser.implementationFiles as file>
    <#include "/@includes/"+file />
</#list>
```
将includes文件夹下的ftl模板中的内容插入进去，ftl模板中写了自定义的方法，返回自定义的SqlNode。
- javacc负责执行Parser.jj生成解析sql语法的类。
- 综上，fmpp是将用户自定义的内容加入到Parser.jj模板中，再使用javacc来生成语法解析器。


- 实现自定义sql语法,创建继承自SqlCall的类,定义自己的sql语法，unparse是主要方法，例如：
```
public class SqlAnalyzeTable extends SqlCall {

	public static final SqlSpecialOperator OPERATOR =
			new SqlSpecialOperator("ANALYZE TABLE", SqlKind.OTHER_DDL);

	private SqlIdentifier tableName;

	private SqlNodeList columnList;

	private boolean withColumns;

	public SqlAnalyzeTable(
			SqlParserPos pos,
			SqlIdentifier tableName,
			SqlNodeList columnList,
			boolean withColumns) {
		super(pos);
		this.tableName = requireNonNull(tableName, "Table name is missing");
		this.columnList = requireNonNull(columnList, "Column list should not be null");
		this.withColumns = withColumns;
	}

	@Override
	public SqlOperator getOperator() {
		return OPERATOR;
	}

	@Override
	public List<SqlNode> getOperandList() {
		return null;
	}

	public SqlNodeList getColumnList() {
		return columnList;
	}

	public void setColumnList(SqlNodeList columnList) {
		this.columnList = columnList;
	}

	public SqlIdentifier getTableName() {
		return tableName;
	}

	public void setTableName(SqlIdentifier tableName) {
		this.tableName = tableName;
	}

	public boolean isWithColumns() {
		return withColumns;
	}

	public void unparse(
			SqlWriter writer,
			int leftPrec,
			int rightPrec) {
		writer.keyword("ANALYZE");
		writer.keyword("TABLE");
		tableName.unparse(writer, leftPrec, rightPrec);
		writer.keyword("COMPUTE");
		writer.keyword("STATISTICS");
		if (withColumns) {
			writer.keyword("FOR");
			writer.keyword("COLUMNS");
			if (!isEmptyList(columnList)) {
				int columnMaxIndex = columnList.size() - 1;
				SqlWriter.Frame columnsFrame = writer.startList(SqlWriter.FrameTypeEnum.SIMPLE);
				for (int idx = 0; idx <= columnMaxIndex; idx++) {
					SqlNode column = columnList.get(idx);
					column.unparse(writer, leftPrec, rightPrec);
					if (idx != columnMaxIndex) {
						writer.sep(",", false);
					}
				}
				writer.endList(columnsFrame);
			}
		}
	}

	public void validate() {
		//todo:
	}
}

```

